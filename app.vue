<script setup lang="ts">
import { appName } from '~/constants'
import Footer from '~/components/Footer.vue'

useHead({
  title: appName,
})
</script>

<template>
  <NuxtLayout>
    <NuxtPage />
    <UNotifications />
    <Footer></Footer>
  </NuxtLayout>
</template>

<style>
html, body , #__nuxt{
  height: 100vh;
  margin: 0;
  padding: 0;
}

html.dark {
  background: #222;
  color: white;
}
</style>
