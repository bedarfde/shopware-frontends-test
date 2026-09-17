<script setup lang="ts">
import type { Schemas } from "#shopware";

// `moorl-twig` renders a Twig template server-side, which a headless storefront
// cannot evaluate. Registering an empty block keeps the CMS layer from logging
// it as unimplemented and from rendering its placeholder.
defineProps<{
  content: Schemas["CmsBlock"];
}>();
</script>

<template>
  <!-- Intentionally renders nothing. -->
</template>
