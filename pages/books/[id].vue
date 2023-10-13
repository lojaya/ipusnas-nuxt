<script setup lang="ts">
import Detail from '~/components/Detail.vue'
import { Book } from '~/constants/interfaces';

const route = useRoute()

const selectedBook = ref<Partial<Book>>({ id: (<any>route.params)?.id })
</script>


<template>
  <UContainer class="flex flex-col h-screen max-w-3xl my-5">
    <div class="flex flex-1 justify-center items-center">
      <div class="flex flex-col items-center gap-5">
        <Detail :book="selectedBook" />
        <UButton to="/" label="Back to Home" class="mb-5"></UButton>
      </div>
    </div>
  </UContainer>
</template>