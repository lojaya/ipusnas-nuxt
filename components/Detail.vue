<script setup lang="ts">
import type { Book, BookItem } from '../constants/interfaces'

const runtimeConfig = useRuntimeConfig()

const props = defineProps<{ book: Partial<Book> }>()
const borrowedBooks = useLocalStorage<any>('ipusnas_borrowed', [])

const emit = defineEmits<{
  (event: 'closeModal'): void,
  (event: 'reloadList'): void
}>()

const { n } = useI18n({
  numberFormats: {
    'id-ID': { currency: { style: 'currency', currency: 'IDR' } }
  }
})

const session = useLocalStorage<any>('ipusnas_session', {})
const isBookCoverOpen = ref(false)
const borrowBookPending = ref(false)
const returnBookPending = ref(false)
const imageLoadError = ref(false)

const source = ref('')
const { text, copy, copied, isSupported } = useClipboard({ source })

const toast = useToast()

const { data: book, pending: bookPending, execute } = await useLazyAsyncData('book', () => $fetch<BookItem>(
  `${runtimeConfig.public.baseUrl}/get-book`, {
  query: {
    book_id: props.book.id,
    access_token: session?.value?.access_token,
  },
}), {
  default: () => { },
  watch: [props.book],
})

const doBorrowBook = async () => {
  if (book.value?.Item?.id) {
    toast.add({
      timeout: 2500,
      title: "Kamu sedang meminjam buku ini",
      color: "red"
    })
  } else {
    const { pending } = useAsyncData('borrowBook', () => $fetch<any>(
      `${runtimeConfig.public.baseUrl}/borrow-book`, {
      method: "POST",
      body: {
        bookId: props.book.id,
        libraryId: book.value?.Library[0].Library.id,
        accessToken: session?.value?.access_token,
      }
    }).then(() => {
      borrowedBooks.value.push(props.book.id)
      useTimeoutFn(execute, 1000)
    }).catch(err => {
      toast.add({
        timeout: 2500,
        title: err?.response?._data?.error_message,
        color: "red"
      })
    }).finally(() => {
      borrowBookPending.value = false
    }))
    borrowBookPending.value = pending.value
  }
}

const doReturnBook = async () => {
  if (!book.value?.Item?.id) {
    toast.add({
      timeout: 2500,
      title: "Kamu belum meminjam buku ini",
      color: "red"
    })
  } else {
    const { pending } = useAsyncData('returnBook', () => $fetch<any>(
      `${runtimeConfig.public.baseUrl}/return-book`, {
      method: "POST",
      body: {
        itemId: book.value?.Item?.id,
        accessToken: session?.value?.access_token,
      }
    }).then(() => {
      useTimeoutFn(execute, 1000)
      emit("reloadList")
    }).catch(err => {
      toast.add({
        timeout: 2500,
        title: err?.response?._data?.error_message,
        color: "red"
      })
    }).finally(() => {
      returnBookPending.value = false
    }))
    returnBookPending.value = pending.value
  }

}
</script>

<template>
  <UCard v-if="!bookPending" :ui="{ divide: 'divide-y divide-gray-100 dark:divide-gray-800' }">
    <template #header>
      <h1 class="font-bold text-center text-lg" @click="emit('closeModal')">{{ book?.Book?.title }}</h1>
      <p class="font-bold text-center text-xs">[{{ book?.Book?.id }}]</p>
    </template>
    <div class="flex flex-col items-center gap-2 pb-5" v-if="book?.Book?.has_book && !isSupported && book?.Item?.pass_zip && book?.Item?.pass_pdf">
      <p class="break-all">Pass Zip: {{ book?.Item?.pass_zip }}</p>
      <p class="break-all">Pass PDF: {{ book?.Item?.pass_pdf }}</p>
    </div>
    <div class="flex flex-col gap-5 items-center sm:flex-row sm:gap-2">
      <img v-if="!imageLoadError" :src="book?.Book?.cover" :alt="book?.Book?.title" @click="isBookCoverOpen = true"
        @error="imageLoadError = true"
        class="w-[300px] sm:min-w-[200px] sm:max-w-[200px] h-full rounded-lg shadow-[0px_0px_5px_-1px_rgba(0,0,0,0.2)]">
      <div v-else
        class="relative bg-gray-100 dark:bg-gray-800 select-none w-full max-w-[200px] sm:min-w-[200px] h-72 rounded-lg shadow-[0px_0px_5px_-1px_rgba(0,0,0,0.2)] border border-solid border-gray-200">
        <div
          class="flex items-center justify-center z-[3] text-gray-800 dark:text-gray-100 h-full w-full text-center font-semibold">
          book cover<br />not available</div>
      </div>
      <div class="flex grow flex-col items-start gap-1.5 text-sm text-left">
        <div class="flex items-center gap-2">
          <strong class="min-w-[90px] text-right">Availability</strong>
          <div class="inline-flex items-center gap-1">
            <strong>{{ book?.Book?.available_copy }} Copies</strong>
            <template v-if="session?.access_token">
              <UButton size="2xs" :loading="borrowBookPending" :color="book?.Book?.available_copy ? 'green' : 'grey'"
              variant="outline" :disabled="!book?.Book?.available_copy" @click="doBorrowBook">Borrow</UButton>
              <UButton size="2xs" color="red" variant="outline" :loading="returnBookPending" @click="doReturnBook">Return
              </UButton>
            </template>
          </div>
        </div>
        <div class="flex items-center gap-2" v-if="book?.Book?.has_book && isSupported && book?.Item?.pass_zip && book?.Item?.pass_pdf">
          <strong class="min-w-[90px] text-right">Password</strong>
          <UButton size="2xs" color="blue" variant="outline" @click="copy(book?.Item?.pass_zip)">
            {{ copied && text === book?.Item?.pass_zip ? 'Copied!' : 'Copy Pass Zip' }}</UButton>
          <UButton size="2xs" color="blue" variant="outline" @click="copy(book?.Item?.pass_pdf)">
            {{ copied && text === book?.Item?.pass_pdf ? 'Copied!' : 'Copy Pass PDF' }}</UButton>
        </div>
        <div class="flex items-start gap-2">
          <strong class="min-w-[90px] text-right">Author</strong>
          <div class="inline-flex items-center gap-1">
            <div>{{ book?.Book?.authors }}</div>
          </div>
        </div>
        <div class="flex items-start gap-2">
          <strong class="min-w-[90px] text-right">ISBN</strong>
          <div class="inline-flex items-center gap-1">
            <div>{{ book?.Book?.isbn }}</div>
          </div>
        </div>
        <div class="flex items-start gap-2">
          <strong class="min-w-[90px] text-right">Format</strong>
          <div class="inline-flex items-center gap-1">
            <div>{{ book?.Book?.extension }} {{ book?.Book?.size }} [{{ n(book?.Book?.price || 0, 'currency')
            }}]</div>
            <UButton v-if="book?.Book?.has_book && book?.Item?.pass_zip && book?.Item?.pass_pdf" size="2xs" color="blue" variant="outline" :to="book?.Item?.out">Download
            </UButton>
          </div>
        </div>
        <div class="flex items-start gap-2">
          <strong class="min-w-[90px] text-right">Category</strong>
          <div class="inline-flex items-center gap-1">
            <div>{{ book?.Category?.name }}</div>
          </div>
        </div>
        <div class="flex items-start gap-2">
          <strong class="min-w-[90px] text-right">Published</strong>
          <div class="inline-flex items-center gap-1">
            <div>{{ useDateFormat(book?.Book?.published_date, 'DD MMMM YYYY').value }}</div>
          </div>
        </div>
        <div class="flex items-start gap-2">
          <strong class="min-w-[90px] text-right">Added On</strong>
          <div class="inline-flex items-center gap-1">
            <div>{{ useDateFormat(book?.Book?.created, 'DD MMMM YYYY, HH:mm:ss').value }}</div>
          </div>
        </div>
        <div class="flex items-start gap-2">
          <strong class="min-w-[90px] text-right">Description</strong>
          <div class="inline-flex items-center gap-1">
            <div class="max-h-24 overflow-scroll" v-html="book?.Book?.description"></div>
          </div>
        </div>
      </div>
    </div>

    <template #footer>
      <div class="flex flex-row gap-1 flex-wrap">
        <UBadge color="gray">rating {{ book?.Statistic?.rating }}</UBadge>
        <UBadge color="gray">reviews {{ book?.Statistic?.total_reviews }}</UBadge>
        <UBadge color="gray">ratings {{ book?.Statistic?.total_ratings }}</UBadge>
        <UBadge color="gray">queues {{ book?.Statistic?.total_queues }}</UBadge>
        <UBadge color="gray">reading {{ book?.Statistic?.total_reading }}</UBadge>
        <UBadge color="gray">wishlist {{ book?.Statistic?.total_wishlists }}</UBadge>
        <UBadge color="gray">has_read {{ book?.Statistic?.total_has_read }}</UBadge>
        <UBadge color="gray">has_borrowed {{ book?.Statistic?.total_has_borrow }}</UBadge>
        <UBadge color="gray">comments {{ book?.Statistic?.total_comments }}</UBadge>
      </div>
    </template>
  </UCard>
  <UCard v-else :ui="{ divide: 'divide-y divide-gray-100 dark:divide-gray-800' }">
    <template #header>
      <USkeleton class="w-full max-w-sm h-4 mx-auto mb-2" />
      <USkeleton class="w-full max-w-xs h-4 mx-auto" />
    </template>
    <USkeleton class="w-full h-72 rounded-lg" />

    <template #footer>
      <div class="flex flex-row gap-1 flex-wrap">
        <USkeleton class="w-20 h-6 mr-1" />
        <USkeleton class="w-24 h-6 mr-1" />
        <USkeleton class="w-16 h-6 mr-1" />
        <USkeleton class="w-24 h-6 mr-1" />
        <USkeleton class="w-20 h-6 mr-1" />
        <USkeleton class="w-32 h-6 mr-1" />
        <USkeleton class="w-28 h-6 mr-1" />
        <USkeleton class="w-16 h-6 mr-1" />
      </div>
    </template>
  </UCard>

  <UModal v-model="isBookCoverOpen" :ui="{ width: 'sm:max-w-[700px]' }">
    <img :src="book?.Book?.cover_origin" alt="" @click="isBookCoverOpen = false" />
  </UModal>
</template>
