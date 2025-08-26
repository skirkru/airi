<script setup lang="ts">
import type { RemovableRef } from '@vueuse/core'

import {
  Alert,
  ProviderAdvancedSettings,
  ProviderBaseUrlInput,
  ProviderBasicSettings,
  ProviderSettingsContainer,
  ProviderSettingsLayout,
} from '@proj-airi/stage-ui/components'
import { useProvidersStore } from '@proj-airi/stage-ui/stores/providers'
import { FieldKeyValues } from '@proj-airi/ui'
import { useDebounceFn } from '@vueuse/core'
import { storeToRefs } from 'pinia'
import { computed, onMounted, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import { useRouter } from 'vue-router'

const { t } = useI18n()
const router = useRouter()
const providersStore = useProvidersStore()
const { providers } = storeToRefs(providersStore) as { providers: RemovableRef<Record<string, any>> }

// Get provider metadata
const providerId = 'ollama'
const providerMetadata = computed(() => providersStore.getProviderMetadata(providerId))

const baseUrl = computed({
  get: () => providers.value[providerId]?.baseUrl || providerMetadata.value?.defaultOptions?.().baseUrl || '',
  set: (value) => {
    if (!providers.value[providerId])
      providers.value[providerId] = {}

    providers.value[providerId].baseUrl = value
  },
})

const headers = ref<{ key: string, value: string }[]>(Object.entries(providers.value[providerId]?.headers || {}).map(([key, value]) => ({ key, value } as { key: string, value: string })) || [{ key: '', value: '' }])

// Validation state
const debounceTime = 500
const isValidating = ref(0)
const isValid = ref(false)
const validationMessage = ref('')

function addKeyValue(headers: { key: string, value: string }[], key: string, value: string) {
  if (!headers)
    return

  headers.push({ key, value })
}

function removeKeyValue(index: number, headers: { key: string, value: string }[]) {
  if (!headers)
    return

  if (headers.length === 1) {
    headers[0].key = ''
    headers[0].value = ''
  }
  else {
    headers.splice(index, 1)
  }
}

// Validation
async function validateConfiguration() {
  if (!providerMetadata.value)
    return

  isValidating.value++
  validationMessage.value = ''
  const startValidationTimestamp = performance.now()
  let finalValidationMessage = ''

  try {
    const config = {
      baseUrl: baseUrl.value.trim(),
      headers: headers.value.filter(header => header.key !== '').reduce((acc, header) => {
        acc[header.key] = header.value
        return acc
      }, {} as Record<string, string>),
    }

    const validationResult = await providerMetadata.value.validators.validateProviderConfig(config)
    isValid.value = validationResult.valid

    if (!isValid.value)
      finalValidationMessage = validationResult.reason
  }
  catch (error) {
    isValid.value = false
    finalValidationMessage = t('settings.dialogs.onboarding.validationError', {
      error: error instanceof Error ? error.message : String(error),
    })
  }
  finally {
    setTimeout(() => {
      isValidating.value--
      validationMessage.value = finalValidationMessage
    }, Math.max(0, debounceTime - (performance.now() - startValidationTimestamp)))
  }
}

const debouncedValidateConfiguration = useDebounceFn(() => {
  if (!baseUrl.value.trim()) {
    isValid.value = false
    validationMessage.value = ''
    isValidating.value = 0
    return
  }
  validateConfiguration()
}, debounceTime)

watch(headers, (newHeaders) => {
  if (newHeaders.length > 0 && (newHeaders[newHeaders.length - 1].key !== '' || newHeaders[newHeaders.length - 1].value !== '')) {
    newHeaders.push({ key: '', value: '' })
  }

  providers.value[providerId].headers = newHeaders.filter(header => header.key !== '').reduce((acc, header) => {
    acc[header.key] = header.value
    return acc
  }, {} as Record<string, string>)
  debouncedValidateConfiguration()
}, {
  deep: true,
  immediate: true,
})

watch(baseUrl, () => {
  debouncedValidateConfiguration()
})

onMounted(() => {
  providersStore.initializeProvider(providerId)

  // Initialize refs with current values
  baseUrl.value = providers.value[providerId]?.baseUrl || providerMetadata.value?.defaultOptions?.().baseUrl || ''

  // Initialize headers if not already set
  if (!providers.value[providerId]?.headers) {
    providers.value[providerId].headers = {}
  }
  if (headers.value.length === 0) {
    headers.value = [{ key: '', value: '' }]
  }

  if (baseUrl.value.trim())
    validateConfiguration()
})

function handleResetSettings() {
  providers.value[providerId] = {
    ...(providerMetadata.value?.defaultOptions as any),
  }
  isValid.value = false
  validationMessage.value = ''
  isValidating.value = 0
}
</script>

<template>
  <ProviderSettingsLayout
    :provider-name="providerMetadata?.localizedName"
    :provider-icon="providerMetadata?.icon"
    :on-back="() => router.back()"
  >
    <ProviderSettingsContainer>
      <ProviderBasicSettings
        :title="t('settings.pages.providers.common.section.basic.title')"
        :description="t('settings.pages.providers.common.section.basic.description')"
        :on-reset="handleResetSettings"
      >
        <ProviderBaseUrlInput
          v-model="baseUrl"
          :placeholder="providerMetadata?.defaultOptions?.().baseUrl as string || ''"
          required
        />
      </ProviderBasicSettings>

      <ProviderAdvancedSettings :title="t('settings.pages.providers.common.section.advanced.title')">
        <FieldKeyValues
          v-model="headers"
          :label="t('settings.pages.providers.common.section.advanced.fields.field.headers.label')"
          :description="t('settings.pages.providers.common.section.advanced.fields.field.headers.description')"
          :key-placeholder="t('settings.pages.providers.common.section.advanced.fields.field.headers.key.placeholder')"
          :value-placeholder="t('settings.pages.providers.common.section.advanced.fields.field.headers.value.placeholder')"
          @add="(key: string, value: string) => addKeyValue(headers, key, value)"
          @remove="(index: number) => removeKeyValue(index, headers)"
        />
      </ProviderAdvancedSettings>

      <!-- Validation Status -->
      <Alert v-if="!isValid && isValidating === 0 && validationMessage" type="error">
        <template #title>
          {{ t('settings.dialogs.onboarding.validationFailed') }}
        </template>
        <template v-if="validationMessage" #content>
          <div class="whitespace-pre-wrap break-all">
            {{ validationMessage }}
          </div>
        </template>
      </Alert>
      <Alert v-if="isValid && isValidating === 0" type="success">
        <template #title>
          {{ t('settings.dialogs.onboarding.validationSuccess') }}
        </template>
      </Alert>
    </ProviderSettingsContainer>
  </ProviderSettingsLayout>
</template>

<route lang="yaml">
  meta:
    layout: settings
    stageTransition:
      name: slide
  </route>
