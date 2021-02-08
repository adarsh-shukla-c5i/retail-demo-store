<template>
  <SecondaryLayout>
    <div class="container">
      <ShopperSelect @profileSwitched="onProfileSwitch"></ShopperSelect>
    </div>
  </SecondaryLayout>
</template>

<script>
import SecondaryLayout from '@/components/SecondaryLayout/SecondaryLayout';
import ShopperSelect from '@/partials/ShopperSelect/ShopperSelect';

export default {
  name: 'ShopperSelectPage',
  components: { SecondaryLayout, ShopperSelect },
  methods: {
    onProfileSwitch() {
      this.$route.push('/');
    },
  },
};
</script>

<style></style>
