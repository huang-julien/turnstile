<script setup lang="ts">
import { useScriptCloudflareTurnstile } from '../composables/turnstile'
import { nextTick, useRuntimeConfig, ref, onMounted, onBeforeUnmount, useElementScriptTrigger } from '#imports'
import type { ElementScriptTrigger } from '#nuxt-scripts'

const props = withDefaults(defineProps<{
  modelValue?: string
  trigger?: ElementScriptTrigger
  element?: string
  siteKey?: string
  options?: Omit<Partial<Turnstile.RenderParameters>, 'callback'>
  resetInterval?: number
}>(), {
  element: 'div',
  options: () => ({}),
  resetInterval: 1000 * 250,
})

const emit = defineEmits<{
  (name: 'update:modelValue', token: string): void
}>()

const config = useRuntimeConfig().public.turnstile

const el = ref()
const unmountStarted = ref(false)
let id: string | undefined | null = undefined
let interval: NodeJS.Timeout
const { render, reset: _reset, remove } = useScriptCloudflareTurnstile({
  scriptOptions: {
    trigger: useElementScriptTrigger({ trigger: props.trigger, el }),
  },
})

const reset = () => {
  if (id) {
    _reset(id)
  }
}
const unmount = () => {
  unmountStarted.value = true
  clearInterval(interval)

  if (id) {
    remove(id)
  }
}

onMounted(async () => {
  await nextTick() // TODO: remove once upstream vue bug is fixed (https://github.com/vuejs/core/issues/5844, https://github.com/nuxt/nuxt/issues/13471)

  id = await render(el.value, {
    sitekey: props.siteKey || config.siteKey,
    ...props.options,
    callback: (token: string) => emit('update:modelValue', token),
  })
  interval = setInterval(reset, props.resetInterval)

  if (unmountStarted.value) {
    unmount()
  }
})

onBeforeUnmount(unmount)

defineExpose({ reset })
</script>

<template>
  <component
    :is="element"
    ref="el"
  />
</template>
