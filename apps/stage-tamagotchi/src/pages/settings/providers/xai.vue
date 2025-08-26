<script setup lang="ts">
import type { RemovableRef } from '@vueuse/core'

import {
  Alert,
  ProviderAdvancedSettings,
  ProviderApiKeyInput,
  ProviderBaseUrlInput,
  ProviderBasicSettings,
  ProviderSettingsContainer,
  ProviderSettingsLayout,
} from '@proj-airi/stage-ui/components'
import { useProvidersStore } from '@proj-airi/stage-ui/stores/providers'
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
const providerId = 'xai'
const providerMetadata = computed(() => providersStore.getProviderMetadata(providerId))

// Use computed properties for settings
const apiKey = computed({
  get: () => providers.value[providerId]?.apiKey || '',
  set: (value) => {
    if (!providers.value[providerId])
      providers.value[providerId] = {}

    providers.value[providerId].apiKey = value
  },
})

const baseUrl = computed({
  get: () => providers.value[providerId]?.baseUrl || '',
  set: (value) => {
    if (!providers.value[providerId])
      providers.value[providerId] = {}

    providers.value[providerId].baseUrl = value
  },
})

// Validation state
const debounceTime = 500
const isValidating = ref(0)
const isValid = ref(false)
const validationMessage = ref('')

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
      apiKey: apiKey.value.trim(),
      baseUrl: baseUrl.value.trim(),
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
  if (!apiKey.value.trim()) {
    isValid.value = false
    validationMessage.value = ''
    isValidating.value = 0
    return
  }
  validateConfiguration()
}, debounceTime)

onMounted(() => {
  // Initialize provider if it doesn't exist
  providersStore.initializeProvider(providerId)

  // Initialize refs with current values
  apiKey.value = providers.value[providerId]?.apiKey || ''
  baseUrl.value = providers.value[providerId]?.baseUrl || ''

  if (apiKey.value.trim())
    validateConfiguration()
})

// Watch settings and update the provider configuration
watch([apiKey, baseUrl], () => {
  providers.value[providerId] = {
    ...providers.value[providerId],
    apiKey: apiKey.value,
    baseUrl: baseUrl.value || '',
  }
  debouncedValidateConfiguration()
}, { deep: true })

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
        <ProviderApiKeyInput
          v-model="apiKey"
          :provider-name="providerMetadata?.localizedName"
          placeholder="xai-..."
        />
      </ProviderBasicSettings>

      <ProviderAdvancedSettings :title="t('settings.pages.providers.common.section.advanced.title')">
        <ProviderBaseUrlInput
          v-model="baseUrl"
          placeholder="https://api.x.ai/v1/"
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
