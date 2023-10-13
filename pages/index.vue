<script setup lang="ts">
import sorts from '../constants/sorts.json'
import categories from '../constants/categories.json'
import type { Book, BookItem } from '../constants/interfaces'
import Entry from '~/components/Entry.vue'
import Detail from '~/components/Detail.vue'
import Notification from '~/components/Notification.vue'

const runtimeConfig = useRuntimeConfig()
const pagingOptions = [24, 48, 64, 96]

// States
const selectedBook = ref<Partial<Book>>({})
const isLoginFormOpen = ref(false)
const isBookDetailOpen = ref(false)
const session = useLocalStorage<any>('ipusnas_session', {})
const borrowedBooks = useLocalStorage<any>('ipusnas_borrowed', [])

const sortOptions = computed(() => sorts.map((obj: any) => {
  if (['borrowing', 'borrowed', 'queued'].includes(obj.id)) {
    if (!session.value?.access_token) return { ...obj, disabled: true }
  }
  return obj
}))

// Selection & Search
const search = ref('')
const selectedSorting = ref(sorts[0])
const selectedCategory = ref(categories[0])

// Pagination
const page = ref(1)
const pageCount = ref(pagingOptions[0])
const pageTotal = ref(200) // This value should be dynamic coming from the API
const pageFrom = computed(() => (page.value - 1) * pageCount.value + 1)
const pageTo = computed(() => Math.min(page.value * pageCount.value, pageTotal.value))

// Toggle/Search Filters
function resetFilters() {
  search.value = ''
  pageCount.value = pagingOptions[0]
  selectedSorting.value = sorts[0]
  selectedCategory.value = categories[0]
}

function openBookDetail(book: Book) {
  isBookDetailOpen.value = true
  selectedBook.value = book
}

function storeSessionToLocalStorage(data: any) {
  isLoginFormOpen.value = false
  session.value = data
  execute()
}

watch(selectedSorting, () => { page.value = 1 })
watch(selectedCategory, (newCategory) => {
  page.value = 1
  if (newCategory.id !== 0) {
    selectedSorting.value = sorts[0]
  }
})

// Data
const { data: books, pending: booksPending, execute } = await useLazyAsyncData('books', () => $fetch<any>(
  `${runtimeConfig.public.baseUrl}/get-books`, {
  query: {
    page: page.value,
    page_size: pageCount.value,
    access_token: session?.value?.access_token,
    sort_by: selectedSorting.value?.id === 'index' ? undefined : selectedSorting.value?.id,
    category_id: selectedCategory.value?.id === 0 ? undefined : selectedCategory.value?.id,
  }
}), {
  transform: (response) => {
    pageTotal.value = Number(response?.meta?.total)
    const data = <BookItem[]>(response?.data)
    if (selectedSorting.value.id == 'borrowed') {
      const bookIds = data.map(book => book.Book.id)
      borrowedBooks.value = useArrayUnique([...bookIds, ...borrowedBooks.value]).value
      console.log(borrowedBooks.value)
    }
    return data
  },
  default: () => [],
  watch: [
    page, pageCount, search,
    selectedSorting, selectedCategory
  ],
})

</script>

<template>
  <UCard class="w-full" :ui="{
    base: '',
    ring: '',
    divide: 'divide-y divide-gray-200 dark:divide-gray-700',
    header: { padding: 'px-4 py-5' },
    body: { padding: '', base: 'divide-y divide-gray-200 dark:divide-gray-700' },
    footer: { padding: 'p-4' },
  }">
    <template #header>
      <div class="flex items-center justify-between gap-3">
        <UButton v-show="session?.access_token" color="red" variant="outline" label="Logout" size="xs"
          icon="i-heroicons-arrow-right-on-rectangle-solid" @click="session = null" trailing />
        <UButton v-show="!session?.access_token" color="green" variant="outline" label="Login" size="xs" :trailing="false"
          icon="i-heroicons-arrow-left-on-rectangle-solid" @click="isLoginFormOpen = true" />
        <h2 class="font-semibold text-xl text-gray-900 dark:text-white text-center">
          Database Ipusnas
        </h2>
        <Notification />
      </div>
    </template>

    <!-- Filters -->
    <div class="flex items-center justify-between gap-3 px-4 py-3">
      <div class="flex items-start flex-col justify-start gap-3 sm:items-center sm:flex-row">
        <UInput v-model="search" icon="i-heroicons-magnifying-glass-20-solid" placeholder="Search..." v-show="false" />
        <UButton icon="i-heroicons-funnel" variant="soft" color="red" @click="resetFilters">Reset</UButton>
      </div>

      <div class="flex items-end flex-col justify-start gap-3 sm:items-center sm:flex-row">
        <USelectMenu v-model="selectedSorting" option-attribute="name" by="id" :options="sortOptions"
          :ui-menu="{ width: 'w-48' }" :ui="{ base: 'disabled:bg-gray-200' }" :disabled="selectedCategory.id !== 0" />
        <USelectMenu v-model="selectedCategory" option-attribute="name" by="id" :options="categories" searchable
          :ui-menu="{ width: 'w-48' }" />
      </div>
    </div>

    <!-- Table -->
    <div class="px-4 py-6 w-full">
      <div class="flex flex-row items-start justify-center flex-wrap gap-1.5">
        <div v-for="item in books" class="md:w-[195px] w-[170px] relative pb-1">
          <Entry :loading="booksPending" :item="item" @open-detail="openBookDetail" :page-type="selectedSorting.id" />
        </div>
      </div>
    </div>

    <!-- Modal -->
    <UModal v-model="isBookDetailOpen" :ui="{ width: 'sm:max-w-[700px]' }">
      <Detail :book="selectedBook" @close-modal="isBookDetailOpen = false" @reload-list="execute" />
    </UModal>

    <UModal v-model="isLoginFormOpen">
      <Login @login-success="storeSessionToLocalStorage" />
    </UModal>

    <!-- Number of rows & Pagination -->
    <template #footer>
      <div class="flex flex-col justify-between items-center gap-4 md:flex-row">
        <div class="flex flex-col items-center gap-4 md:items-start md:gap-1">
          <span class="inline-flex items-center gap-1 text-sm leading-5">
            Showing
            <span class="font-medium">{{ pageFrom }}</span> to
            <span class="font-medium">{{ pageTo }}</span> of
            <span class="font-medium">{{ pageTotal }}</span> results
          </span>
          <div class="inline-flex items-center gap-1 text-sm leading-5">
            <span>by</span>
            <USelect v-model="pageCount" :options="pagingOptions" class="w-16" size="xs" />
            <span>per page</span>
          </div>
        </div>
        <UPagination v-model="page" :max="5" :page-count="pageCount" :total="pageTotal" size="xs" :ui="{
          wrapper: 'flex items-center gap-2',
          rounded: '!rounded-full min-w-[32px] justify-center',
        }" />
      </div>
    </template>
  </UCard>
</template>
