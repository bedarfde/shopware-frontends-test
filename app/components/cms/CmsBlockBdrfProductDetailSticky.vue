<script setup lang="ts">
import { useCmsBlock } from "#imports";
import type { Schemas } from "#shopware";

// `bdrf-product-detail-sticky` is a shop-specific block, so it is absent from
// the generated Store API schema. It combines what the stock product page
// splits across `gallery-buybox` and `product-description-reviews`: the buy box
// stays pinned while gallery and description scroll past it.
type CmsBlockBdrfProductDetailSticky = Schemas["CmsBlock"] & {
  type: "bdrf-product-detail-sticky";
  slots: (Schemas["CmsSlot"] & { slot: "gallery" | "description" | "buy-box" })[];
};

const props = defineProps<{
  content: CmsBlockBdrfProductDetailSticky;
}>();

const { getSlotContent } = useCmsBlock(props.content);
const galleryContent = getSlotContent("gallery");
const descriptionContent = getSlotContent("description");
const buyBoxContent = getSlotContent("buy-box");
</script>

<template>
  <div
    class="cms-block-bdrf-product-detail-sticky w-full flex flex-col lg:flex-row justify-center items-start gap-4 lg:gap-10 lg:px-0 overflow-clip"
  >
    <div class="w-full lg:w-3/5 min-w-0 flex flex-col gap-10">
      <CmsGenericElement :content="galleryContent" />
      <CmsGenericElement :content="descriptionContent" />
    </div>
    <!-- `top-4` matches the container gap; there is no fixed header to clear.
         The container clips with `overflow-clip` rather than `overflow-hidden`
         (as `gallery-buybox` does) because the latter would create a scroll
         container and silently disable this `sticky`. -->
    <div class="w-full lg:w-2/5 min-w-0 lg:sticky lg:top-4">
      <CmsGenericElement :content="buyBoxContent" />
    </div>
  </div>
</template>
