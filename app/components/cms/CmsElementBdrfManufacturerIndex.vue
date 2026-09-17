<script setup lang="ts">
import type { Schemas } from "#shopware";

// `bdrf-manufacturer-index` is a shop-specific element, so it is absent from
// the generated Store API schema. Only the slot type is narrowed for now; the
// `config`/`data` shapes get typed once this element actually renders them.
type CmsElementBdrfManufacturerIndex = Schemas["CmsSlot"] & {
  type: "bdrf-manufacturer-index";
};

defineProps<{
  content: CmsElementBdrfManufacturerIndex;
}>();
</script>

<template>
  <div class="cms-element-bdrf-manufacturer-index" />
</template>
