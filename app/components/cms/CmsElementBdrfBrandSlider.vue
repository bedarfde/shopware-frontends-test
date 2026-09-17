<script setup lang="ts">
import type { ElementConfig } from "@shopware/composables";
import { getTranslatedProperty } from "@shopware/helpers";
import { computed } from "vue";

import { useCmsElementConfig } from "#imports";
import type { Schemas } from "#shopware";

type BrandSliderItem = {
  mediaId: string;
  mediaUrl: string;
};

type BrandSliderConfig = {
  speed?: ElementConfig<number>;
  logoHeight?: ElementConfig<number>;
  sliderItems?: ElementConfig<BrandSliderItem[]>;
};

type BrandSliderData = {
  apiAlias: "array_struct";
  slides?: Schemas["Media"][];
  speed?: number;
  logoHeight?: number;
};

// `bdrf-brand-slider` is a shop-specific element, so it is absent from the
// generated Store API schema. `data` is omitted from the base type rather than
// intersected onto it: `CmsSlot["data"]` is a recursive `GenericRecord` that
// `Media[]` does not satisfy.
type CmsElementBdrfBrandSlider = Omit<Schemas["CmsSlot"], "config" | "data"> & {
  type: "bdrf-brand-slider";
  config: BrandSliderConfig;
  data?: BrandSliderData | null;
};

type BrandSlide = {
  key: string;
  src: string;
  alt: string;
};

const props = defineProps<{
  content: CmsElementBdrfBrandSlider;
}>();

const DEFAULT_SPEED_SECONDS = 60;
const DEFAULT_LOGO_HEIGHT_PX = 40;
/** Repeat the logos until one set is wide enough that the loop never gaps. */
const MIN_ITEMS_PER_SET = 8;

/**
 * `getConfigValue` returns `false` — not `undefined` — for mapped fields, so
 * every numeric read goes through this guard before falling back.
 */
const toPositiveNumber = (value: unknown): number | undefined =>
  typeof value === "number" && Number.isFinite(value) && value > 0 ? value : undefined;

// Narrow to the config alone, like SwSlider does, so the element's own `data`
// shape does not have to satisfy the composable's `CmsSlot` constraint.
const { getConfigValue } = useCmsElementConfig({
  config: props.content.config,
} as Omit<Schemas["CmsSlot"], "config"> & { config: BrandSliderConfig });

/** Seconds for one full loop of the strip. */
const durationSeconds = computed(
  () =>
    toPositiveNumber(getConfigValue("speed")) ??
    toPositiveNumber(props.content.data?.speed) ??
    DEFAULT_SPEED_SECONDS,
);

const logoHeight = computed(
  () =>
    toPositiveNumber(getConfigValue("logoHeight")) ??
    toPositiveNumber(props.content.data?.logoHeight) ??
    DEFAULT_LOGO_HEIGHT_PX,
);

// Resolved media carries translated alt/title; the raw config only has URLs, so
// it is a fallback for backends that do not resolve this element's data.
const slides = computed<BrandSlide[]>(() => {
  const media = props.content.data?.slides ?? [];

  if (media.length) {
    return media.map((item) => ({
      key: item.id,
      src: item.url,
      alt: getTranslatedProperty(item, "alt") || getTranslatedProperty(item, "title"),
    }));
  }

  const items = getConfigValue("sliderItems");
  if (!Array.isArray(items)) return [];

  return items.map((item) => ({
    key: item.mediaId,
    src: item.mediaUrl,
    // No alt is available here; a partner logo strip is decorative, and an
    // empty alt beats announcing a file name.
    alt: "",
  }));
});

const marqueeItems = computed<BrandSlide[]>(() => {
  const base = slides.value;
  if (!base.length) return [];

  const repeats = Math.ceil(MIN_ITEMS_PER_SET / base.length);
  return Array.from({ length: repeats }, () => base).flat();
});
</script>

<template>
  <div v-if="marqueeItems.length" class="cms-element-bdrf-brand-slider w-full overflow-hidden">
    <!-- Two identical sets on a `w-max` track: translating by -50% advances
         exactly one set width, so the loop wraps without a jump. -->
    <div
      class="brand-slider-track flex w-max items-center"
      :style="{
        '--brand-slider-duration': `${durationSeconds}s`,
        '--brand-slider-logo-height': `${logoHeight}px`,
      }"
    >
      <ul
        v-for="set in 2"
        :key="set"
        class="flex shrink-0 items-center gap-12 pr-12"
        :aria-hidden="set === 2 ? 'true' : undefined"
      >
        <li v-for="(slide, index) in marqueeItems" :key="`${slide.key}-${index}`" class="shrink-0">
          <NuxtImg
            preset="productDetail"
            loading="lazy"
            class="brand-slider-logo w-auto max-w-none object-contain"
            :src="slide.src"
            :alt="slide.alt"
            :height="logoHeight * 2"
          />
        </li>
      </ul>
    </div>
  </div>
</template>

<style scoped>
.brand-slider-track {
  min-height: var(--brand-slider-logo-height, 40px);
  animation: brand-slider-scroll var(--brand-slider-duration, 60s) linear infinite;
  will-change: transform;
}

.brand-slider-logo {
  height: var(--brand-slider-logo-height, 40px);
}

@keyframes brand-slider-scroll {
  from {
    transform: translateX(0);
  }

  to {
    transform: translateX(-50%);
  }
}

@media (prefers-reduced-motion: reduce) {
  .brand-slider-track {
    animation: none;
    transform: none;
  }
}
</style>
