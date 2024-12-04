<script setup lang="ts">
import type { BoxLayout, DisplayMode } from "@shopware-pwa/composables-next";
import {
  getProductName,
  getProductFromPrice,
  getBiggestThumbnailUrl,
} from "@shopware-pwa/helpers-next";
import type { Schemas } from "#shopware";

const { pushSuccess, pushError } = useNotifications();
const { t } = useI18n();
const { getErrorsCodes } = useCartNotification();
const localePath = useLocalePath();
const { formatLink } = useInternationalization(localePath);
const { resolveCartError } = useCartErrorParamsResolver();

const props = withDefaults(
  defineProps<{
    product: Schemas["Product"];
    layoutType?: BoxLayout;
    displayMode?: DisplayMode;
  }>(),
  {
    layoutType: "standard",
    displayMode: "standard",
  },
);
const { product } = toRefs(props);
const { addToCart, isInCart, count } = useAddToCart(product);

const addToCartProxy = async () => {
  await addToCart();
  const errors = getErrorsCodes();

  errors?.forEach((element) => {
    const { messageKey, params } = resolveCartError(element);
    pushError(t(`errors.${messageKey}`, params as Record<string, unknown>));
  });

  if (!errors.length)
    pushSuccess(
      t(`cart.messages.addedToCart`, { p: props.product?.translated.name }),
    );
};

const fromPrice = getProductFromPrice(props.product);

const imageElement = useTemplateRef("imageElement");
const { height } = useElementSize(imageElement);

const DEFAULT_THUMBNAIL_SIZE = 10;
function roundUp(num: number) {
  return num ? Math.ceil(num / 100) * 100 : DEFAULT_THUMBNAIL_SIZE;
}

const srcPath = computed(() => {
  return `${getBiggestThumbnailUrl(
    product.value.cover?.media,
  )}?&height=${roundUp(height.value)}&fit=cover`;
});

// extract tooltip entries from customFields
const tooltipConfig = computed(() => {
  if (!product.value) return;

  const hasTooltipEntries = Object.keys(product.value?.customFields || {}).some(
    (key) => key.includes("tooltip"),
  );

  if (!hasTooltipEntries) return;

  const entries = Object.entries(product.value?.customFields || {}).map(
    ([key, value]) => {
      if (key.includes("tooltip")) {
        return {
          key: key
            .replace("tooltip_one", "1")
            .replace("tooltip_two", "2")
            .replace("tooltip_three", "3"),
          value,
        };
      }
    },
  );

  return entries;
});

const getTooltipForNumber = (number: number) => {
  if (!tooltipConfig.value) return;

  return tooltipConfig.value.find((entry) =>
    entry?.key?.includes(number.toString()),
  )?.value;
};

const tooltipModalController = useModal();
const getCurrentTooltip = ref("");

const openTooltip = (number: number) => {
  getCurrentTooltip.value = getTooltipForNumber(number);
  tooltipModalController.open();
};
</script>

<template>
  <div
    class="sw-product-card group relative max-w-full inline-block bg-white"
    data-testid="product-box"
  >
    <CrossSellVariantConfigurator v-if="product.id" :product="product" :key="product.id" />
    <div class="px-4 pb-4">
      <div class="flex items-center justify-between">
        <div class="">
          <SharedListingProductPrice
            :product="product"
            class="ml-auto"
            data-testid="product-box-product-price"
          />
        </div>

        <button
          v-if="!fromPrice"
          type="button"
          class="justify-center py-2 px-2 border border-transparent shadow-sm text-sm font-medium rounded-md flex"
          :class="{
            'text-white bg-black hover:bg-gray-700': !isInCart,
            'text-secondary-600 bg-secondary-100': isInCart,
            'opacity-50 cursor-not-allowed': !product.available,
          }"
          data-testid="add-to-cart-button"
          :disabled="!product.available"
          @click="addToCartProxy"
        >
          {{ $t("Select this bundle") }}
          <div v-if="isInCart" class="flex ml-2">
            <div class="w-5 h-5 i-carbon-shopping-bag text-secondary-600" />
            {{ count }}
          </div>
        </button>
      </div>
    </div>
    <div :class="['w-full rounded-md overflow-hidden']">
      <div class="absolute top-5 -left-1 z-10">
        <span
          v-if="product.markAsTopseller"
          class="bg-[#FFBD5D] px-2.5 py-1.5 color-black text-xl"
        >
          {{ $t("product.badges.topseller") }}
        </span>
      </div>
      <button
        v-if="getTooltipForNumber(1)"
        class="absolute bg-white border-black border border-3 hover:bg-black hover:text-white text-black text-3xl rounded-full w-14 h-14 flex items-center justify-center"
        :style="{ top: `30%`, right: `25%` }"
        @click.prevent="openTooltip(1)"
      >
        1
      </button>
      <button
        v-if="getTooltipForNumber(2)"
        class="absolute bg-white border-black border border-3 hover:bg-black hover:text-white text-black text-3xl rounded-full w-14 h-14 flex items-center justify-center"
        :style="{ top: `50%`, left: `20%` }"
        @click.prevent="openTooltip(2)"
      >
        2
      </button>
      <button
        v-if="getTooltipForNumber(3)"
        class="absolute bg-white border-black border border-3 hover:bg-black hover:text-white text-black text-3xl rounded-full w-14 h-14 flex items-center justify-center"
        :style="{ bottom: `20%`, right: `40%` }"
        @click.prevent="openTooltip(3)"
      >
        3
      </button>
      <img
        ref="imageElement"
        :src="srcPath"
        :alt="getProductName({ product }) || ''"
        :class="{
          'w-full h-full': true,
          'object-cover':
            displayMode === 'cover' ||
            (displayMode === 'standard' && layoutType === 'image'),
          'object-contain': displayMode === 'contain',
          'object-scale-down':
            displayMode === 'standard' && layoutType !== 'image',
        }"
        data-testid="product-box-img"
      />
    </div>
    <SharedModal :controller="tooltipModalController">{{
      getCurrentTooltip
    }}</SharedModal>
  </div>
</template>
