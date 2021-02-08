<template>
  <Layout :isLoading="isLoading" :previousPageLinkProps="previousPageLinkProps">
    <template #default>
      <div class="container text-left">
        <div class="row">
          <div class="col">
            <div class="row">
              <div class="col mb-3 mb-sm-5">
                <div class="quantity-readout p-3 font-weight-bold">{{ cartQuantityReadout }}</div>
              </div>
            </div>

            <ul class="cart-items">
              <CartItem
                v-for="item in cart.items"
                :key="item.product_id"
                :product_id="item.product_id"
                :quantity="item.quantity"
                :cartPrice="item.price"
                class="mb-4"
              ></CartItem>
            </ul>
          </div>
          <div v-if="cart.items.length > 0" class="summary-container col-lg-auto">
            <div class="summary p-4">
              <div class="summary-quantity">{{ summaryQuantityReadout }}</div>
              <div class="summary-total mb-2 font-weight-bold">Your Total: {{ formattedCartTotal }}</div>
              <router-link to="/checkout" class="checkout-btn mb-3 btn btn-outline-dark btn-block btn-lg"
                >Checkout</router-link
              >
              <AbandonCartButton class="abandon-cart"></AbandonCartButton>
            </div>
          </div>
        </div>
        <RecommendedProductsSection :experiment="experiment" :recommendedProducts="relatedProducts" :feature="feature">
          <template #heading
            >Compare similar items
            <!-- <DemoGuideBadge
              v-if="demoGuideBadgeArticle"
              :article="demoGuideBadgeArticle"
              hideTextOnSmallScreens
            ></DemoGuideBadge> -->
          </template>
        </RecommendedProductsSection>
      </div>
    </template>
  </Layout>
</template>

<script>
import swal from 'sweetalert';

import { mapState, mapGetters, mapActions} from 'vuex';
import { RepositoryFactory } from '@/repositories/RepositoryFactory';

import { AnalyticsHandler } from '@/analytics/AnalyticsHandler';

import CartItem from './components/CartItem.vue';
import Layout from '@/components/Layout/Layout';
import AbandonCartButton from '@/partials/AbandonCartButton/AbandonCartButton';
import RecommendedProductsSection from '@/components/RecommendedProductsSection/RecommendedProductsSection';
import { discountProductPrice } from '@/util/discountProductPrice';

import { getDemoGuideArticleFromPersonalizeARN } from '@/partials/AppModal/DemoGuide/config';

const RecommendationsRepository = RepositoryFactory.get('recommendations');
const MAX_RECOMMENDATIONS = 6;
const EXPERIMENT_FEATURE = 'product_detail_related';


export default {
  name: 'Cart',
  components: {
    Layout,
    CartItem,
    AbandonCartButton,
    RecommendedProductsSection,
  },
  data() {
    return {
      quantity: 1,
      feature: EXPERIMENT_FEATURE,
      relatedProducts: null,
      experiment: null,
    };
  },
  created() {
    AnalyticsHandler.cartViewed(this.user, this.cart, this.cartQuantity, this.cartTotal);
  },
  computed: {
    ...mapState({
      cart: (state) => state.cart.cart,
      user: (state) => state.user,
      lastVisitedPage: (state) => state.lastVisitedPage.route,
    }),
    ...mapGetters(['cartQuantity', 'cartTotal', 'formattedCartTotal']),
    isLoading() {
      return !this.cart;
    },
    previousPageLinkProps() {
      if (!this.lastVisitedPage) return null;

      return { text: 'Continue Shopping', to: this.lastVisitedPage };
    },
    cartQuantityReadout() {
      if (this.cartQuantity === null) return null;

      return `(${this.cartQuantity}) ${this.cartQuantity === 1 ? 'item' : 'items'} in your cart shopping cart`;
    },
    summaryQuantityReadout() {
      if (this.cartQuantity === null) return null;

      return `Summary (${this.cartQuantity}) ${this.cartQuantity === 1 ? 'item' : 'items'}`;
    },
  },
  watch: {
    $route: {
      immediate: true,
      handler() {
        this.fetchData();
      },
    },
    personalizeUserID() {
      this.getRelatedProducts();
    },
  },
  methods: {
    ...mapActions(['addToCart']),
    resetQuantity() {
      this.quantity = 1;
    },
    async addProductToCart() {
      await this.addToCart({
        product: {
          ...this.product,
          price: this.discount ? discountProductPrice(this.product.price) : this.product.price,
        },
        quantity: this.quantity,
        feature: this.$route.query.feature,
        exp: this.$route.query.exp,
      });

      this.renderAddedToCartConfirmation();

      this.resetQuantity();
    },
    async fetchData() {
      // await this.getProductByID(this.$route.params.id);

      this.getRelatedProducts();

      // this.recordProductViewed(this.$route.query.feature, this.$route.query.exp, this.$route.query.di);
    },
    async getRelatedProducts() {
      // reset in order to trigger recalculation in carousel - carousel UI breaks without this
      this.relatedProducts = null;

      this.experiment = null;
      this.demoGuideBadgeArticle = null;

      const response = await RecommendationsRepository.getRelatedProducts(
        this.personalizeUserID ?? '',
        this.cart.items[0].product_id,
        // this.cart.items[0].product_id,
        MAX_RECOMMENDATIONS,
        EXPERIMENT_FEATURE,
      );

      if (response.headers) {
        const experimentName = response.headers['x-experiment-name'];
        const personalizeRecipe = response.headers['x-personalize-recipe'];

        if (experimentName) this.experiment = `Active experiment: ${experimentName}`;
        if (personalizeRecipe) this.demoGuideBadgeArticle = getDemoGuideArticleFromPersonalizeARN(personalizeRecipe);
      }

      this.relatedProducts = response.data;

      if (this.relatedProducts.length > 0 && 'experiment' in this.relatedProducts[0]) {
        AnalyticsHandler.identifyExperiment(this.user, this.relatedProducts[0].experiment);
      }
    },
    renderAddedToCartConfirmation() {
      swal({
        title: 'Added to Cart',
        icon: 'success',
        buttons: {
          cancel: 'Continue Shopping',
          cart: 'View Cart',
        },
      }).then((value) => {
        switch (value) {
          case 'cancel':
            break;
          case 'cart':
            this.$router.push('/cart');
        }
      });
    },
  },
};
</script>

<style scoped>
.quantity-readout {
  border-radius: 4px;
  background: var(--grey-200);
  font-size: 1.15rem;
}

.cart-items {
  list-style-type: none;
  padding: 0;
}

.summary {
  border: 1px solid var(--grey-300);
  border-radius: 2px;
}

.summary-total {
  font-size: 1.15rem;
}

.summary-quantity {
  font-size: 1.15rem;
}

.checkout-btn {
  border-color: var(--grey-900);
  border-width: 2px;
  font-size: 1rem;
}

.checkout-btn:hover,
.checkout-btn:focus {
  background: var(--grey-900);
}

@media (min-width: 768px) {
  .quantity-readout {
    font-size: 1.75rem;
  }

  .summary-total {
    font-size: 1.5rem;
  }

  .summary-quantity {
    font-size: 1.5rem;
  }

  .checkout-btn {
    font-size: 1.25rem;
  }
}

@media (min-width: 992px) {
  .summary-container {
    min-width: 350px;
  }

  .summary {
    position: sticky;
    top: 120px;
  }

  .abandon-cart {
    max-width: 400px;
  }
}
</style>
