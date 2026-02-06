<template>
  <div
    class="sticky top-0 z-10 flex flex-shrink-0 items-center justify-center overflow-auto overflow-x-auto whitespace-nowrap bg-primary p-4"
  >
    <AppShortcutsPrompt v-if="response == null && !isEmbed" class="flex-1" />

    <div v-if="response == null && isEmbed">
      <HoppButtonSecondary
        :label="`${t('app.documentation')}`"
        to="https://docs.hoppscotch.io/documentation/features/rest-api-testing#response"
        :icon="IconExternalLink"
        blank
        outline
        reverse
      />
    </div>

    <div v-else-if="response" class="flex flex-1 flex-col">
      <div
        v-if="response.type === 'loading' || isLoading"
        class="flex flex-col items-center justify-center"
      >
        <HoppSmartSpinner class="my-4" />
        <span class="text-secondaryLight">{{ t("state.loading") }}</span>
      </div>

      <component
        :is="response.component"
        v-if="response.type === 'extension_error'"
        class="flex-1"
      />
      <HoppSmartPlaceholder
        v-if="response.type === 'interceptor_error'"
        :src="`/images/states/${colorMode.value}/upload_error.svg`"
        :alt="
          response.error?.humanMessage?.heading?.(t) || t('error.network_fail')
        "
        :heading="
          response.error?.humanMessage?.heading?.(t) || t('error.network_fail')
        "
        :text="
          response.error?.humanMessage?.description?.(t) ||
          t('error.network_fail')
        "
      >
        <template #body>
          <div
            v-if="
              response.error &&
              typeof response.error === 'object' &&
              'error' in response.error &&
              getErrorDetails(response.error.error)
            "
            class="mb-2 w-full overflow-auto whitespace-normal rounded bg-primaryLight px-4 py-2 font-mono text-red-400"
          >
            {{ getErrorDetails(response.error.error) }}
          </div>
          <AppKernelInterceptor
            class="rounded border border-dividerLight p-2"
          />
        </template>
      </HoppSmartPlaceholder>
      <HoppSmartPlaceholder
        v-if="response.type === 'network_fail'"
        :src="`/images/states/${colorMode.value}/upload_error.svg`"
        :alt="`${t('error.network_fail')}`"
        :heading="t('error.network_fail')"
        :text="t('helpers.network_fail')"
      >
        <template #body>
          <div
            v-if="response.error && getErrorDetails(response.error)"
            class="mb-2 w-full overflow-auto whitespace-normal rounded bg-primaryLight px-4 py-2 font-mono text-red-400"
          >
            {{ getErrorDetails(response.error) }}
          </div>
          <AppKernelInterceptor
            class="rounded border border-dividerLight p-2"
          />
        </template>
      </HoppSmartPlaceholder>
      <HoppSmartPlaceholder
        v-if="response.type === 'script_fail'"
        :src="`/images/states/${colorMode.value}/upload_error.svg`"
        :alt="`${t('error.script_fail')}`"
        :label="t('error.script_fail')"
        :text="t('helpers.script_fail')"
      >
        <template #body>
          <div
            class="mt-2 w-full overflow-auto whitespace-normal rounded bg-primaryLight px-4 py-2 font-mono text-red-400"
          >
            {{ response.error.name }}: {{ response.error.message }}<br />
            {{ response.error.stack }}
          </div>
        </template>
      </HoppSmartPlaceholder>
      <div
        v-if="
          (response.type === 'success' || response.type === 'fail') &&
          !isLoading
        "
        class="flex items-center text-tiny font-semibold"
      >
        <div
          :class="statusCategory.className"
          class="inline-flex flex-1 space-x-4"
        >
          <span v-if="response.statusCode">
            <span class="text-secondary"> {{ t("response.status") }}: </span>
            {{ `${response.statusCode}\xA0 • \xA0`
            }}{{
              getStatusCodeReasonPhrase(
                response.statusCode,
                response.statusText
              )
            }}
          </span>
          <span v-if="response.meta && response.meta.responseDuration">
            <span class="text-secondary"> {{ t("response.time") }}: </span>
            {{ `${response.meta.responseDuration} ms` }}
          </span>
          <span
            v-if="response.meta && response.meta.responseSize"
            v-tippy="
              readableResponseSize
                ? { theme: 'tooltip' }
                : { onShow: () => false }
            "
            :title="`${response.meta.responseSize} B`"
          >
            <span class="text-secondary"> {{ t("response.size") }}: </span>
            {{
              readableResponseSize
                ? readableResponseSize
                : `${response.meta.responseSize} B`
            }}
          </span>
        </div>
      </div>
    </div>
    <AppInspection
      v-if="response?.type !== 'loading'"
      :inspection-results="tabResults"
      :class="[
        response === null || response?.type === 'network_fail'
          ? 'absolute right-2 top-2'
          : '-m-2 ml-2',
      ]"
    />
  </div>
</template>

<script setup lang="ts">
import { computed } from "vue"
import findStatusGroup from "@helpers/findStatusGroup"
import type { HoppRESTResponse } from "~/helpers/types/HoppRESTResponse"
import { useI18n } from "@composables/i18n"
import { useColorMode } from "@composables/theming"
import { getStatusCodeReasonPhrase } from "~/helpers/utils/statusCodes"
import { useService } from "dioc/vue"
import { InspectionService } from "~/services/inspection"
import { RESTTabService } from "~/services/tab/rest"
import IconExternalLink from "~icons/lucide/external-link"

const t = useI18n()
const colorMode = useColorMode()
const tabs = useService(RESTTabService)

const props = withDefaults(
  defineProps<{
    response: HoppRESTResponse | null | undefined
    isEmbed?: boolean
    isLoading?: boolean
  }>(),
  {
    isLoading: false,
  }
)

/**
 * Gives the response size in a human readable format
 * (changes unit from B to MB/KB depending on the size)
 * If no changes (error res state) or value can be made (size < 1KB ?),
 * it returns undefined
 */
const readableResponseSize = computed(() => {
  if (
    props.response === null ||
    props.response === undefined ||
    props.response.type === "loading" ||
    props.response.type === "network_fail" ||
    props.response.type === "script_fail" ||
    props.response.type === "fail" ||
    props.response.type === "extension_error"
  )
    return undefined

  const size = props.response.meta.responseSize

  if (size >= 100000) return (size / 1000000).toFixed(2) + " MB"
  if (size >= 1000) return (size / 1000).toFixed(2) + " KB"

  return undefined
})

const statusCategory = computed(() => {
  if (
    props.response === null ||
    props.response === undefined ||
    props.response.type === "loading" ||
    props.response.type === "network_fail" ||
    props.response.type === "script_fail" ||
    props.response.type === "fail" ||
    props.response.type === "extension_error"
  )
    return {
      name: "error",
      className: "text-red-500",
    }
  return findStatusGroup(props.response.statusCode)
})

const inspectionService = useService(InspectionService)

/**
 * Extracts detailed error information from the error object
 * for better debugging and user feedback
 */
const getErrorDetails = (error: unknown): string | null => {
  if (!error) return null

  // Handle Error instances
  if (error instanceof Error) {
    return `${error.name}: ${error.message}`
  }

  // Handle RelayError objects
  if (typeof error === "object" && error !== null) {
    const relayError = error as any

    // Check for kind-based errors (RelayError structure)
    if ("kind" in relayError && "message" in relayError) {
      let details = `[${relayError.kind.toUpperCase()}] ${relayError.message}`

      // Add cause if available
      if (relayError.cause) {
        const causeStr =
          typeof relayError.cause === "string"
            ? relayError.cause
            : relayError.cause instanceof Error
              ? relayError.cause.message
              : JSON.stringify(relayError.cause)
        details += `\nCause: ${causeStr}`
      }

      // Add phase for timeout errors
      if (relayError.kind === "timeout" && relayError.phase) {
        details += `\nPhase: ${relayError.phase}`
      }

      return details
    }

    // Handle generic error objects with message property
    if ("message" in relayError) {
      return String(relayError.message)
    }
  }

  // Fallback for other types
  if (typeof error === "string") {
    return error
  }

  // Last resort: stringify the error
  try {
    return JSON.stringify(error, null, 2)
  } catch {
    return String(error)
  }
}
const tabResults = inspectionService.getResultViewFor(
  tabs.currentTabID.value,
  (result) => result.locations.type === "response"
)
</script>
