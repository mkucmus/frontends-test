<script setup lang="ts">
import type {
  CmsElementCrossSelling,
  SliderElementConfig,
} from "@shopware-pwa/composables-next";
import { ref, computed, useTemplateRef } from "vue";
import { useElementSize } from "@vueuse/core";
import type { Schemas } from "#shopware";

const props = defineProps<{
  content: CmsElementCrossSelling;
}>();

const currentTabIndex = ref<number>(0);

const crossSellCollections = computed(() => {
  return (
    props.content?.data?.crossSellings?.filter(
      (collection) => !!collection.products.length,
    ) || []
  );
});

const toggleTab = (index: number) => {
  if (currentTabIndex.value === index) return;
  currentTabIndex.value = index;
  // load first product from the collection
  loadProduct(crossSellCollections.value[index]?.products[0]);
};

// on ssr load the first one
const crossSellProduct = ref(
  crossSellCollections.value[currentTabIndex.value]?.products[0],
);
const loadProduct = (product: Schemas["Product"]) => {
  crossSellProduct.value = product;
};
</script>

<template>
  <div class="md:flex">
    <ul
      class="flex-column flex-basis-[400px] space-y space-y-4 text-sm font-medium text-gray-500 dark:text-gray-400 md:me-4 mb-4 md:mb-0 list-none"
    >
      <li v-for="(collection, index) of crossSellCollections">
        <h4
          aria-current="page"
          :key="index"
          :class="{
            ' text-white bg-black rounded-lg active dark:bg-blue-600 hover:text-white':
              currentTabIndex === index,
            ' bg-gray-50 hover:bg-gray-100 dark:bg-gray-800 dark:hover:bg-gray-700 dark:hover:text-white':
              currentTabIndex !== index,
          }"
          class="inline-flex items-center px-4 py-3 rounded-lg hover:text-gray-900 w-full"
          @click="toggleTab(index)"
        >
          {{ collection.crossSelling.name }}
        </h4>
        <transition name="fade" mode="out-in">
          <ul
            class="ps-5 mt-2 space-y-1 list-inside list-none"
            v-if="currentTabIndex === index"
          >
            <li
              class="p-2 cursor-pointer hover:text-black"
              :class="{ 'text-black': crossSellProduct?.id === product.id }"
              v-for="product of collection.products"
              @click="loadProduct(product)"
            >
              {{ product.name }}
            </li>
          </ul>
        </transition>
      </li>
    </ul>

    <div
      class="p-6 bg-gray-50 text-medium text-gray-500 dark:text-gray-400 dark:bg-gray-800 rounded-lg w-full"
    >
      <transition name="fade" mode="out-in">
        <div v-if="crossSellProduct?.id">
          <h3 class="text-lg font-bold text-gray-900 dark:text-white mb-2">
            {{ crossSellProduct.name }}
          </h3>
          <ProductCardCrossSell v-if="crossSellProduct.id" :product="crossSellProduct" :id="crossSellProduct.id" />
        </div>
      </transition>
    </div>
  </div>
</template>

<style scoped>
.fade-enter-active,
.fade-leave-active {
  transition: all 0.2s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

.cms-element-cross-selling {
  display: flex;
}

.cms-element-cross-selling .flex-col {
  width: 200px;
}

.cms-element-cross-selling .flex-1 {
  flex: 1;
}
</style>
