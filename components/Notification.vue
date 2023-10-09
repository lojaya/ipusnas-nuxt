<script setup lang="ts">
const runtimeConfig = useRuntimeConfig()

const TEN_MINUTES = 1000 * 60 * 10
const session = useLocalStorage<any>('ipusnas_session', {})

const { data: notifications, pending: notifPending, execute } = await useLazyAsyncData('notifications', () => $fetch<any>(
  `${runtimeConfig.public.baseUrl}/notifications`, {
  query: {
    page: 1,
    page_size: 20,
    access_token: session?.value?.access_token,
  }
}), {
  transform: (response) => {
    return <any[]>(response?.data.map((v: any) => ([v])))
  },
  default: () => []
})

onMounted(() => {
  useIntervalFn(() => {
    execute()
  }, TEN_MINUTES)
})

</script>

<template>
  <UDropdown :items="notifications" mode="hover" :popper="{ placement: 'bottom-start' }" :ui="{ width: 'w-80' }">
    <UButton color="white" variant="soft" :loading="notifPending" :trailing-icon="notifPending ? '' : 'i-heroicons-bell'" />
    <template #item="{ item }">
      <p class="text-left font-medium text-gray-900 dark:text-white">
        {{ item.message }} <span class="text-xs">({{ item.elapsed_time }})</span>
      </p>
    </template>
  </UDropdown>
</template>
