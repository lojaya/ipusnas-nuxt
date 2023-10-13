<script setup lang="ts">
import type { Book, BookItem } from '../constants/interfaces'

const props = defineProps<{ item: BookItem, loading: Boolean, pageType: string }>()
const borrowedBooks = useLocalStorage<any>('ipusnas_borrowed', [])

const emit = defineEmits<{
  (event: 'openDetail', data: Book): void
}>()

const isBorrowed = computed<boolean>(() => {
  return borrowedBooks.value.includes(props.item.Book.id) && !['borrowing', 'borrowed'].includes(props.pageType)
})
</script>

<template>
  <div class="cursor-pointer flex flex-col items-center gap-2" @click="() => emit('openDetail', item.Book)">
    <template v-if="loading">
      <USkeleton class="w-full h-72 rounded-lg" />
      <USkeleton class="w-44 h-3" />
      <USkeleton class="w-36 h-3" />
    </template>
    <template v-else>
      <div class="w-full relative h-72 overflow-hidden rounded-lg shadow-[0px_0px_5px_-1px_rgba(0,0,0,0.2)]" :class="{
        'hover:shadow-[0px_0px_10px_2px_rgba(0,0,0,0.2)]': true,
        'hover:opacity-90 duration-150 hover:duration-150': true,
        // 'h-72': item.Book.coverLoaded === false,
        'max-h-72': item.Book.coverLoaded === true,
        'h-auto': item.Book.coverLoaded === true
      }
        ">
        <div v-if="isBorrowed" class="absolute w-full h-full bg-black opacity-60 z-[2]">
          <div class="flex items-center justify-center z-[3] text-white h-full w-full font-semibold">borrowed</div>
        </div>
        <img v-if="!item.Book.coverNonExists" @load="() => item.Book.coverLoaded = true" :src="item.Book.cover"
          class="w-full rounded-lg" @error="item.Book.coverNonExists = true">
        <div v-else
          class="relative bg-gray-100 dark:bg-gray-800 w-full h-72 select-none rounded-lg border border-solid border-gray-200">
          <div
            class="flex items-center justify-center z-[3] text-gray-800 dark:text-gray-100 h-full w-full text-center font-semibold">
            book cover<br />not available</div>
        </div>
      </div>
      <h2 class="font-bold text-xs text-center">{{ item.Book.title }}</h2>
    </template>
  </div>
</template>
