<script setup lang="ts">
import { FormError, FormSubmitEvent } from '@nuxt/ui/dist/runtime/types'

const runtimeConfig = useRuntimeConfig()

const state = ref({
  email: undefined,
  password: undefined
})

const toast = useToast()

const validateForm = (state: any): FormError[] => {
  const errors = []
  if (!state.email) errors.push({ path: 'email', message: 'Required' })
  if (!state.password) errors.push({ path: 'password', message: 'Required' })
  return errors
}

async function submitForm(event: FormSubmitEvent<any>) {
  $fetch<any>(`${runtimeConfig.public.baseUrl}/login`, {
    method: "POST",
    body: {
      username: state.value.email,
      password: state.value.password,
    },
  }).then(res => {
    emit("loginSuccess", res)
  }).catch(err => {
    toast.add({
      timeout: 2500,
      title: err?.response?._data?.error_message,
      color: "red"
    })
  })
}

const emit = defineEmits<{
  (event: 'loginSuccess', data: any): void
}>()
</script>

<template>
  <UCard>
    <UForm :validate="validateForm" :state="state" @submit="submitForm">
      <UFormGroup label="Email" name="email">
        <UInput v-model="state.email" />
      </UFormGroup>
      <UFormGroup label="Password" name="password">
        <UInput v-model="state.password" type="password" />
      </UFormGroup>
      <UButton type="submit" class="mt-4">
        Submit
      </UButton>
    </UForm>
  </UCard>
</template>
