<template>
  <BPopover ref="popover" v-bind="computedProps" v-model="modelValue" tooltip>
    <template v-if="slots.default" #default>
      <slot />
    </template>
    <template v-if="slots.target" #target="scope">
      <slot name="target" v-bind="scope" />
    </template>
    <template v-if="slots.title" #title>
      <slot name="title" />
    </template>
  </BPopover>
</template>

<script setup lang="ts">
import {computed, ref} from 'vue'
import BPopover from '../BPopover/BPopover.vue'
import type {BPopoverProps, BTooltipProps} from '../../types/ComponentProps'
import {useDefaults} from '../../composables/useDefaults'

const _props = withDefaults(defineProps<Omit<BTooltipProps, 'modelValue'>>(), {
  click: undefined,
  teleportDisabled: undefined,
  teleportTo: undefined,
  content: undefined,
  customClass: undefined,
  delay: undefined,
  floatingMiddleware: undefined,
  hide: undefined,
  html: undefined,
  id: undefined,
  inline: undefined,
  interactive: undefined,
  manual: undefined,
  noAutoClose: undefined,
  noFade: undefined,
  noFlip: undefined,
  noHide: undefined,
  noShift: undefined,
  noninteractive: undefined,
  offset: undefined,
  placement: undefined,
  realtime: undefined,
  reference: undefined,
  strategy: undefined,
  target: undefined,
  title: undefined,
  variant: undefined,
})

const props = useDefaults(_props, 'BTooltip')

const slots = defineSlots<{
  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  default?: (props: Record<string, never>) => any
  target?: (props: {
    show: () => void
    hide: (e: Event) => void
    toggle: (e: Event) => void
    showState: boolean
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
  }) => any
  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  title?: (props: Record<string, never>) => any
}>()

const modelValue = defineModel<BTooltipProps['modelValue']>({default: undefined})

const computedProps = computed<BPopoverProps>(() => {
  const {interactive, noninteractive, ...rest} = props
  return {noninteractive: noninteractive !== undefined ? noninteractive : !interactive, ...rest}
})

const popover = ref<null | InstanceType<typeof BPopover>>(null)

defineExpose({
  hide: popover.value?.hide,
  show: popover.value?.show,
  toggle: popover.value?.toggle,
})
</script>
