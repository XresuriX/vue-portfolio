<script setup>
import Layout from "../Layout.vue";

const currentYear = new Date().getFullYear()

</script>

<template>
  <Layout class="max-w-full py-5 border-t dark:border-darkBorderColor dark:bg-darkBgColor border-borderColor">
    <p class="text-[0.9rem] text-disableColor dark:text-darkDisableColor text-center">Copyright © {{ currentYear }} by Leslie Taffe.</p>
  </Layout>
</template>