<script>
  import Breadcrumbs from '../../components/Breadcrumbs'
  import Select from '../../components/Select'
  import ChoiceSlider from '../../components/ChoiceSlider.vue'
  import ProductCard from '../../components/ProductCard/ProductCard'
  import ReadMore from '@/components/ReadMore'
  import ProductLightbox from '@/pages/Product/ProductLightbox'

  /*
  * @bug: Description is cut with the tags in its string.
  * */
  export default {
    name: 'p-product',
    components: {
      'c-product-lightbox': ProductLightbox,
      'c-select': Select,
      'c-breadcrumbs': Breadcrumbs,
      'c-read-more': ReadMore,
      'choice-slider': ChoiceSlider,
      'c-product-card': ProductCard
    },
    props: {
      data: {
        type: Object,
        required: true,
        validator (data) {
          return data && Object.keys(data).length
        }
      }
    },
    data () {
      return {
        currentImageId: 1,
        count: 1,
        size: null,
        currentColors: null,
        lightboxOpened: false,
        addingToCart: false,
        userFavorited: null
      }
    },
    jsonld () {
      return {
        '@context': 'http://schema.org',
        '@type': 'Product',
        'name': `${this.data.product.name}`,
        'description': `${this.data.product.description}`,
        'image': `https://shop.city-seller.ru${this.currentImage.url}`,
        'offers': {
          '@type': 'Offer',
          'price': `${this.data.product.price}`,
          'priceCurrency': 'RUB'
        },
        'additionalProperty': {
          '@type': 'PropertyValue',
          'name': 'Страна-производитель, модель',
          'value': `${this.data.made_in ? this.data.made_in.name : ''}, ${this.data.product.sku}`
        }
      }
    },
    computed: {
      inFavorite () {
        if (this.userFavorited === null) {
          return this.data.product.favorite
        }

        return this.userFavorited
      },
      currentImage () {
        const {currentImageId} = this,
          {photos: productImages} = this.data

        if (!currentImageId || currentImageId === -1) {
          return null
        }

        const resultImage = productImages.find(({id}) => id === currentImageId)

        if (!resultImage) {
          return null
        }

        return resultImage
      },
      currentImageStyle () {
        return this.img(this.currentImage.url)
      },
      price () {
        const {product} = this.data

        if (!this.$isPresent(product)) {
          return null
        }

        return parseInt(product.price, 10)
      },
      sizes () {
        const {options} = this.data,
          result = []

        if (!options) {
          return result
        }

        for (const optionKey in options) {
          if (options.hasOwnProperty(optionKey)) {
            const data = options[optionKey].size,
              intIndex = data.short.findIndex(shortEl => shortEl === 'INT')

            if (intIndex !== -1) {
              result.push({
                name: data.values[intIndex],
                value: data.values[intIndex],
                sku_id: data.sku_id,
                id: optionKey
              })
            }
            else {
              result.push({
                name: data.values[0],
                value: data.values[0],
                sku_id: data.sku_id,
                id: optionKey
              })
            }
          }
        }

        return result
      },
      relatedSizes () {
        const {size} = this,
          result = {}

        if (!size) {
          return null
        }

        const properSizeBranch = this.data.options[size.id].size,
          excludeIndex = properSizeBranch.short.findIndex(shortEl => shortEl === 'INT')

        for (let index = 0; index < properSizeBranch.short.length; index++) {
          if (index !== excludeIndex && Object.keys(result).length !== 3) {
            result[properSizeBranch.short[index]] = properSizeBranch.values[index]
          }
        }

        return result
      },
      currentColor () {
        const {product} = this.data,
          result = product.colors.split(';')

        return result[0]
      },
      relatedColors () {
        return this.data.related
      },
      option () {
        if (this.size) {
          return this.data.options[Number(this.size.id)].size.sku_id
        }

        return 0
      },
      eyezonVisible () {
        return Boolean(this.data.eyezon.length)
      },
      lightboxImages () {
        const photosCopy = JSON.parse(JSON.stringify(this.data.photos)),
          toBeFirstPhotoIndex = photosCopy.findIndex(photo => photo.id === this.currentImageId),
          toBeFirstPhoto = photosCopy.splice(toBeFirstPhotoIndex, 1)[0]

        photosCopy.unshift(toBeFirstPhoto)

        return photosCopy
      },

      isSizeInCart () {
        for (const prod of this.$store.state.cart.products) {
          if (prod.option === this.size.sku_id) {
            return true
          }
        }

        return false
      }
    },
    created () {
      this.currentImageId = this.data.photos[0].id

      if (this.sizes.length) {
        this.size = this.sizes[0]
      }
    },
    methods: {
      setCurrentImageId (newValue) {
        if (this.currentImageId !== newValue) {
          this.currentImageId = newValue
        }
      },
      setCount (newValue) {
        if (!newValue || newValue < 0) {
          this.count = 1
        }
        else {
          this.count = newValue
        }
      },
      openFastBuyModal () {
        if (this.size) {
          return this.$modal({
            name: 'm-fast-buy-modal',
            data: {
              sku_id: this.size.sku_id
            }
          })
        }
      },
      async addProductToCart () {
        if (this.size !== null && !this.isSizeInCart) {
          const size = this.size.id

          Number(size)

          this.addingToCart = true

          if (!this.$store.getters['user/isAuthorized']) {
            if (this.$cookies.get('_cs_cart_id')) {
              const cart_id = this.$cookies.get('_cs_cart_id')
              const {data} = await this.$axios.post('cart', {option: this.option, cart_id})

              if (data.data.added) {
                const {data: cartData} = await this.$axios.get('cart', {params: {cart_id}})

                this.$store.commit('cart/SET_CART_PRODUCTS', cartData.data.products)
              }
            }
            else {
              const {data} = await this.$axios.post('/cart', {option: this.option}),
                {cart} = data.data

              if (data.data.added) {
                const {data: cartData} = await this.$axios.get('cart', {params: {cart_id: cart}})

                this.$store.commit('cart/SET_CART_PRODUCTS', cartData.data.products)
              }

              this.$cookies.set('_cs_cart_id', cart, {
                path: '/',
                expires: new Date(Date.now() * 2),
                sameSite: true
              })
            }
          }
          else {
            const cart_id = null

            const {data} = await this.$axios.post('/cart', {option: this.option, cart_id})

            if (data.data.added) {
              const {data: cartData} = await this.$axios.get('cart')

              this.$store.commit('cart/SET_CART_PRODUCTS', cartData.data.products)
            }
          }


          this.addingToCart = false
        }
      },
      openEyezon () {
        const title = `${this.data.product.name}(${this.data.product.sku}, Белый)`

        window.eyezonAPI.function(this.data.eyezon.join(','), title)
      },
      openLightbox () {
        this.lightboxOpened = true
      },
      async addToFavorites () {
        const {colors, sku} = this.data.product,
          payload = {colors, sku},
          favListCookie = this.$cookies.get('_cs_fav_list')

        if (!this.$store.getters['user/isAuthorized'] && favListCookie) {
          payload.fav_list = favListCookie
        }
        else {
          payload.fav_list = null
        }

        if (!this.inFavorite) {
          const {data: favoriteData} = await this.$axios.$post('favorites', payload),
            {list} = favoriteData

          if (!favListCookie) {
            this.$cookies.set('_cs_fav_list', list, {
              path: '/',
              expires: new Date(Date.now() * 2),
              sameSite: true
            })
          }

          this.userFavorited = true
        }
        else {
          await this.$axios.$delete('favorites', {
            params: payload
          })

          this.userFavorited = false
        }
      }
    }
  }
</script>

<template>
  <div class="p-product">
    <c-product-lightbox v-model="lightboxOpened"
                        :data="lightboxImages"/>
    <div class="l-wrapper p-product__container">
      <div v-if="$isPresent(data.breadcrumbs)"
           class="p-product__breadcrumbs">
        <c-breadcrumbs :data="data.breadcrumbs"/>
      </div>
      <div class="p-product__fav"
           @click="addToFavorites">
        <span class="icon-ilem-liked">
          <span class="path1"/>
          <span class="path2"
                :class="{'icon-ilem-liked-done': inFavorite}"/>
        </span>
      </div>
      <div v-if="$isPresent(data.product)"
           class="g-row">
        <div class="g-col-md-8">
          <div :style="currentImageStyle"
               class="p-product__img"
               @click="openLightbox">
            <div :style="{backgroundColor: currentColor}"
                 class="p-product__img-color"/>
            <div class="p-product__img-labels">
              <div class="p-product__img-label p-product__img-label_black">
                {{ data.product.collection || 'Basic' }}
              </div>
              <div v-if="data.product.sale"
                   class="p-product__img-label p-product__img-label_discount">
                — {{ Number(data.product.sale) }}%
              </div>
            </div>
          </div>
          <div class="g-row p-product__img-previews">
            <div v-if="eyezonVisible"
                 class="g-col-4">
              <div class="p-product__img-preview p-product__img-preview_live mb-30"
                   @click="openEyezon">
                <div class="p-product__img-preview_live_wrap">
                  <div class="p-product__img-preview-content">
                    <img src="~assets/img/icons/eyezon.svg"
                         class="p-product__img-eyezon">
                    <img src="~assets/img/icons/cip.svg"
                         class="p-product__img-cip">
                  </div>
                  <span class="p-product__img-preview-text">
                    Живой просмотр
                  </span>
                </div>
              </div>
            </div>
            <div v-for="photo in data.photos"
                 :key="photo.id"
                 class="g-col-4">
              <div :style="img(photo.thumb)"
                   class="p-product__img-preview mb-30"
                   @click="setCurrentImageId(photo.id)"/>
            </div>
          </div>
        </div>
        <div class="g-offset-lg-1 g-col-md-8 g-col-lg-7">
          <div v-if="$isPresent(data.brands)"
               class="t-p-semibold p-product__brand">
            {{ data.brands[0].name }}
          </div>
          <div class="t-cap p-product__name">
            {{ data.product.name }}
          </div>
          <div class="p-product__select-size-wrap">
            <div class="p-product__select-size-el t-cap">
              Выбрать размер:
            </div>
            <div class="p-product__select-size-el">
              <c-select id="sizeSelect"
                        v-model="size"
                        :items="sizes"
                        class="p-product__select"
                        placeholder="Выберите размер"/>
            </div>
          </div>
          <div v-if="size && relatedSizes && Object.keys(relatedSizes).length"
               class="p-product__sizes-wrap">
            <div class="p-product__sizes-el t-cap">
              Соответствие размера:
            </div>
            <div class="p-product__sizes-el">
              <div class="p-product__sizes">
                <div v-for="(sizeValue, sizeKey) in relatedSizes"
                     :key="sizeKey"
                     class="p-product__size">
                  {{ sizeKey }} <strong>
                  {{ sizeValue }}
                </strong>
                </div>
              </div>
            </div>
          </div>
          <div v-if="relatedColors.length"
               class="p-product__colors-wrap">
            <div class="p-product__colors-el t-cap">
              Есть другие цвета:
            </div>
            <div class="p-product__colors-el">
              <nuxt-link v-for="(relatedColor,colorIndex) in relatedColors"
                         :key="colorIndex"
                         :to="relatedColor.link"
                         class="p-product__color"
                         :style="{backgroundColor: relatedColor.color}"/>
            </div>
          </div>
          <div v-if="false"
               class="p-product__count-wrap">
            <div class="p-product__count-el t-cap">
              Количество:
            </div>
            <div class="p-product__count-el">
              <span class="p-product__count-control"
                    @click="setCount(count - 1)">
                -
              </span>
              <label class="c-input p-product__count-input">
                <input v-model.number="count"
                       class="c-input__input"
                       type="number">
              </label>
              <span class="p-product__count-control"
                    @click="setCount(count + 1)">
                +
              </span>
            </div>
          </div>
          <div class="p-product__price-wrap">
            <div class="p-product__price-el">
              <span v-if="false"
                    class="p-product__price-cap">
                от
              </span>
              <span class="p-product__price">
                {{ mFmt(Math.floor(((price * (100 - data.product.sale)) / 100.0)), true) }}
              </span>
            </div>
            <div v-if="data.product.sale"
                 class="p-product__price-el">
              <span class="p-product__price p-product__price_old">
                {{ mFmt(price, true) }}
              </span>
            </div>
          </div>
          <div class="p-product__buttons">
            <button class="c-button p-product__button p-product__button_add"
                    :disabled="addingToCart"
                    :class="{'c-button_green': !isSizeInCart,'loading': addingToCart}"
                    @click="addProductToCart">
              <span>
                {{ isSizeInCart ? 'В корзине' : 'Добавить в корзину' }}
              </span>
            </button>
            <button class="c-button c-button_normal p-product__button p-product__button_buy"
                    @click="openFastBuyModal">
              <span>
                Быстрая покупка
              </span>
            </button>
          </div>
          <div class="p-product__deliveries">
            <template v-if="false">
              <span class="g-link g-link_underline_line">
                Доставим
                <div class="p-product__delivery-popup">
                  <div class="p-product__delivery-popup-title t-p-semibold">
                    СДЭК
                  </div>
                  <div class="p-product__delivery-popup-desc">
                    Приедет в течение недели, (зависит от города) <br>
                    Стоимость от 890 руб.
                  </div>
                  <div class="p-product__delivery-popup-title mt-20 t-p-semibold">
                    DHL
                  </div>
                  <div class="p-product__delivery-popup-desc">
                    Приедет в течение 2 недель (зависит от города) <br>
                    Стоимость от 1490 руб.
                  </div>
                </div>
              </span> по Москве <strong>
              сегодня
            </strong>
            </template>
            Стоимость доставки - <strong>
            бесплатно
          </strong>
            <template v-if="false">
              <span class="g-link g-link_underline_dashed">
                Доставка в другие города
                <div class="p-product__delivery-popup">
                  <div class="p-product__delivery-popup-title t-p-semibold">
                    СДЭК
                  </div>
                  <div class="p-product__delivery-popup-desc">
                    Приедет в течение недели, (зависит от города) <br>
                    Стоимость от 890 руб.
                  </div>
                  <div class="p-product__delivery-popup-title mt-20 t-p-semibold">
                    DHL
                  </div>
                  <div class="p-product__delivery-popup-desc">
                    Приедет в течение 2 недель (зависит от города) <br>
                    Стоимость от 1490 руб.
                  </div>
                </div>
              </span>
            </template>
          </div>
          <p class="p-product__payment t-cap">
            Оплата Банковской картой, PayPal, Сбербанк Онлайн <br>
            <strong>
              Предоплата 100%
            </strong>
          </p>
          <div class="p-product__chars t-cap">
            <div v-if="data.made_in"
                 class="p-product__char p-product__char_half">
              <div v-if="data.made_in.image"
                   class="p-product__char-icon outline">
                <img :src="imgNoBack('/img/flags/' + data.made_in.image)">
              </div>
              <div class="p-product__char-txt">
                Сделано в {{ data.made_in.name_made }}
              </div>
            </div>
            <div v-if="data.designed_in"
                 class="p-product__char p-product__char_half">
              <div v-if="data.designed_in.image"
                   class="p-product__char-icon outline">
                <img :src="imgNoBack('/img/flags/' + data.designed_in.image)">
              </div>
              <div class="p-product__char-txt">
                Дизайн из {{ data.designed_in.name_design }}
              </div>
            </div>
            <div class="p-product__char p-product__char_full">
              <div class="p-product__char-icon p-product__char-icon_gray">
                <span class="icon-article"/>
              </div>
              <div class="p-product__char-txt">
                Артикул <strong>
                {{ data.product.id }}
              </strong> / Артикул производителя
                <strong>
                  {{ data.product.sku }}
                </strong>
              </div>
            </div>
          </div>
          <div v-if="data.product.description"
               class="p-product__desc p-product__desc-more t-cap">
            <c-read-more :text="data.product.description"
                         :maxChars="240"/>
          </div>
        </div>
      </div>
      <!--<div class="p-product__section">
        <div class="g-row">
          <div class="g-offset-md-1 g-col-md-15">
            <div class="t-h1">
              Дополнить образ
            </div>
          </div>
        </div>
        <choice-slider class="p-product__section-content"></choice-slider>
      </div>-->
      <div v-if="$isPresent(data.also)"
           class="p-product__section">
        <div class="g-row">
          <div class="g-offset-md-1 g-col-md-15">
            <div class="t-h2">
              Вам также понравится
            </div>
          </div>
        </div>
        <choice-slider class="p-product__section-content"
                       :products="data.also"/>
      </div>
      <div v-if="$isPresent(data.history)"
           class="p-product__section">
        <div class="g-row">
          <div class="g-offset-md-1 g-col-md-15">
            <div class="t-h2">
              Просмотрено ранее
            </div>
          </div>
        </div>
        <div class="g-row p-product__section-content">
          <c-product-card v-for="item in data.history"
                          :key="item.image"
                          :data="item"
                          class="mb-30 g-col-16 p-product__viewed-item"/>
        </div>
      </div>
    </div>
  </div>
</template>

<style lang="scss" src="./product.scss"/>