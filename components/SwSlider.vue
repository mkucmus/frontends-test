<script setup lang="ts">
import type { SliderElementConfig } from "@shopware-pwa/composables-next";
import type { Schemas } from "#shopware";
import {
  computed,
  onBeforeUnmount,
  onMounted,
  ref,
  useSlots,
  useTemplateRef,
  watch,
  defineProps,
  defineEmits,
} from "vue";
import type { CSSProperties, VNodeArrayChildren } from "vue";
import { useElementSize, useResizeObserver } from "@vueuse/core";

const props = withDefaults(
  defineProps<{
    config: SliderElementConfig;
    slidesToShow?: number;
    slidesToScroll?: number;
    gap?: string;
    autoplay?: boolean;
    autoplaySpeed?: number;
    activeTab?: number;
  }>(),
  {
    slidesToShow: 1,
    slidesToScroll: 1,
    gap: "0px",
    autoplay: false,
    autoplaySpeed: 3000,
    activeTab: 0,
  },
);

const { getConfigValue } = useCmsElementConfig({
  config: props.config,
} as Schemas["CmsSlot"] & {
  config: SliderElementConfig;
});

const slots = useSlots();
const childrenRaw = computed(
  () => (slots?.default?.()[0].children as VNodeArrayChildren) ?? [],
);

const slidesToScroll = computed(() =>
  props.slidesToScroll >= props.slidesToShow
    ? props.slidesToShow
    : props.slidesToScroll,
);
const slidesToShow = computed(() =>
  props.slidesToShow >= childrenRaw.value.length
    ? childrenRaw.value.length
    : props.slidesToShow,
);
const children = computed<string[]>(() => {
  if (childrenRaw.value.length === 0) return [];
  return [
    ...childrenRaw.value.slice(-slidesToShow.value),
    ...childrenRaw.value,
    ...childrenRaw.value.slice(0, slidesToShow.value),
  ] as string[];
});
const emit = defineEmits<{
  (e: "changeSlide", index: number): void;
  (e: "changeTab", index: number): void;
}>();
const slider = useTemplateRef<HTMLElement>("slider");
const imageSlider = useTemplateRef<HTMLElement>("imageSlider");
const imageSliderTrackStyle = ref<CSSProperties>();
const activeSlideIndex = ref<number>(0);
const speed = ref<number>(300);
const imageSliderTrack = useTemplateRef<HTMLElement>("imageSliderTrack");
const autoPlayInterval = ref();
const isReady = ref<boolean>();
const isSliding = ref<boolean>();
const activeTabIndex = ref(props.activeTab);

const { width: imageSliderWidth } = useElementSize(imageSlider);
let timeoutGuard: ReturnType<typeof setTimeout> | undefined;

onMounted(() => {
  initSlider();

  useResizeObserver(slider, () => {
    clearTimeout(timeoutGuard);
    timeoutGuard = setTimeout(() => {
      buildImageSliderTrackStyle(activeSlideIndex.value);
    }, 100);
  });
});

onBeforeUnmount(() => {
  clearInterval(autoPlayInterval.value);
});

watch(
  () => props.autoplay && isReady.value,
  (value) => {
    if (value) {
      autoPlayInterval.value = setInterval(() => {
        next();
      }, props.autoplaySpeed);
    } else {
      if (autoPlayInterval.value) {
        clearInterval(autoPlayInterval.value);
      }
    }
  },
  {
    immediate: true,
  },
);

const imageSliderStyle = computed(() => {
  if (getConfigValue("displayMode") === "cover") {
    return {
      height: getConfigValue("minHeight"),
      margin: `0 -${props.gap}`,
    };
  } else {
    return {
      minHeight: getConfigValue("minHeight"),
    };
  }
});

const verticalAlignValue = computed(
  () => getConfigValue("verticalAlign") || "flex-start",
);
const displayModeValue = computed(
  () => getConfigValue("displayMode") || "standard",
);

const navigationArrowsValue = computed(
  () => props.config?.navigationArrows?.value || "none",
);
const navigationDotsValue = computed(
  () => props.config?.navigationDots?.value || "none",
);

function initSlider() {
  if (imageSlider.value) {
    setTimeout(() => {
      buildImageSliderTrackStyle(activeSlideIndex.value, false, undefined);
      isReady.value = true;
    }, 100);
  }
}

function buildImageSliderTrackStyle(
  transformIndex: number,
  moving: boolean = false,
  callback = () => {},
) {
  let styleObj: CSSProperties = {
    transform: `translate3d(-${
      (transformIndex + slidesToShow.value) *
      (imageSliderWidth.value / slidesToShow.value)
    }px, 0px, 0px)`,
    width: `${children.value.length * imageSliderWidth.value}px`,
  };

  if (imageSliderTrackStyle.value?.height) {
    styleObj.height = imageSliderTrackStyle.value?.height;
  }

  if (moving) {
    styleObj = {
      ...styleObj,
      transition: `transform ${speed.value}ms ease 0s`,
    };
    imageSliderTrackStyle.value = { ...styleObj };
    isSliding.value = true;
    setTimeout(() => {
      delete styleObj.transition;
      imageSliderTrackStyle.value = { ...styleObj };
      isSliding.value = false;
      callback();
    }, speed.value);
  } else {
    imageSliderTrackStyle.value = { ...styleObj };
  }

  setTimeout(() => {
    let height = "unset";
    if (displayModeValue.value === "cover") {
      height = "100%";
    } else if (displayModeValue.value === "standard") {
      const childComponent =
        imageSliderTrack.value?.children[transformIndex + 1];
      // If image exist
      height = childComponent?.children[0].children[0].clientHeight
        ? `${childComponent.clientHeight}px`
        : (height = `auto`);
    }
    styleObj = {
      ...styleObj,
      height,
    };
    imageSliderTrackStyle.value = { ...styleObj };
  });
}

function next() {
  if (isSliding.value) return;
  activeSlideIndex.value = activeSlideIndex.value + slidesToScroll.value;
  buildImageSliderTrackStyle(activeSlideIndex.value, true, () => {
    if (
      activeSlideIndex.value ===
      children.value.length - slidesToShow.value * 2
    ) {
      activeSlideIndex.value = 0;
      buildImageSliderTrackStyle(activeSlideIndex.value);
    }
    emit("changeSlide", activeSlideIndex.value);
  });
}

function previous() {
  if (isSliding.value) return;
  activeSlideIndex.value = activeSlideIndex.value - slidesToScroll.value;
  buildImageSliderTrackStyle(activeSlideIndex.value, true, () => {
    if (activeSlideIndex.value <= 0 - slidesToShow.value) {
      activeSlideIndex.value = children.value.length - slidesToShow.value * 3;
      buildImageSliderTrackStyle(activeSlideIndex.value);
    }
    emit("changeSlide", activeSlideIndex.value);
  });
}

function goToSlide(index: number) {
  if (isSliding.value) return;
  if (activeSlideIndex.value === index) return;
  activeSlideIndex.value = index;
  buildImageSliderTrackStyle(activeSlideIndex.value, true);
  emit("changeSlide", activeSlideIndex.value);
}

function goToTab(index: number) {
  activeTabIndex.value = index;
  emit("changeTab", index);
}

defineExpose({
  next,
  previous,
  goToSlide,
  goToTab,
});
</script>
<template>
  <div>
    <div class="tabs">
      <button
        v-for="(child, index) in childrenRaw"
        :key="index"
        :class="{ active: index === activeTabIndex }"
        @click="goToTab(index)"
      >
         <ProductCard
            v-for="product of crossSellCollections?.[currentTabIndex]?.products"
            :key="product.id"
            class="w-[300px]"
            :product="product"
            :layout-type="getConfigValue('boxLayout')"
            :display-mode="getConfigValue('displayMode')"
          />
      </button>
    </div>
    <div class="tab-content">
      <component :is="childrenRaw[activeTabIndex]" />
    </div>
  </div>
</template>

<style scoped>
.tabs {
  display: flex;
  gap: 10px;
}

.tabs button {
  padding: 10px;
  cursor: pointer;
}

.tabs button.active {
  font-weight: bold;
  border-bottom: 2px solid #000;
}

.tab-content {
  margin-top: 20px;
}
</style>
