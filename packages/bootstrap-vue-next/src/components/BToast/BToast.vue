<template>
  <BTransition
    :no-fade="props.noFade"
    v-bind="props.transProps"
    @before-enter="onBeforeEnter"
    @after-enter="onAfterEnter"
    @after-leave="onAfterLeave"
  >
    <div
      v-if="isToastVisible"
      :id="props.id"
      ref="element"
      class="toast"
      :class="[props.toastClass, computedClasses]"
      tabindex="0"
      :role="!isToastVisible ? undefined : props.isStatus ? 'status' : 'alert'"
      :aria-live="!isToastVisible ? undefined : props.isStatus ? 'polite' : 'assertive'"
      :aria-atomic="!isToastVisible ? undefined : true"
    >
      <component
        :is="props.headerTag"
        v-if="$slots.title || props.title"
        class="toast-header"
        :class="props.headerClass"
      >
        <slot name="title" :hide="hideFn">
          <strong class="me-auto">
            {{ props.title }}
          </strong>
        </slot>
        <BCloseButton v-if="!props.noCloseButton" @click="hideFn('close')" />
      </component>
      <template v-if="$slots.default || props.body">
        <component
          :is="computedTag"
          class="toast-body"
          style="display: block"
          :class="props.bodyClass"
          v-bind="computedLinkProps"
          @click="computedLink ? hideFn() : () => {}"
        >
          <slot :hide="hideFn">
            {{ props.body }}
          </slot>
        </component>
      </template>
      <BProgress
        v-if="typeof modelValue === 'number' && props.progressProps !== undefined"
        :animated="props.progressProps.animated"
        :precision="props.progressProps.precision"
        :show-progress="props.progressProps.showProgress"
        :show-value="props.progressProps.showValue"
        :striped="props.progressProps.striped"
        :variant="props.progressProps.variant"
        :max="modelValue"
        :value="remainingMs"
        height="4px"
      />
    </div>
  </BTransition>
</template>

<script setup lang="ts">
import {computed, ref, watch, watchEffect} from 'vue'
import {useBLinkHelper} from '../../composables/useBLinkHelper'
import type {BToastProps} from '../../types/ComponentProps'
import BTransition from '../BTransition.vue'
import BCloseButton from '../BButton/BCloseButton.vue'
import BLink from '../BLink/BLink.vue'
import BProgress from '../BProgress/BProgress.vue'
import {BvTriggerableEvent} from '../../utils'
import {useCountdown} from '../../composables/useCountdown'
import {useColorVariantClasses} from '../../composables/useColorVariantClasses'
import {useDefaults} from '../../composables/useDefaults'
import {useCountdownHover} from '../../composables/useCountdownHover'

const _props = withDefaults(defineProps<Omit<BToastProps, 'modelValue'>>(), {
  bgVariant: null,
  body: undefined,
  bodyClass: undefined,
  headerClass: undefined,
  headerTag: 'div',
  id: undefined,
  interval: 'requestAnimationFrame',
  isStatus: false,
  noCloseButton: false,
  noFade: false,
  noHoverPause: false,
  noResumeOnHoverLeave: false,
  progressProps: undefined,
  showOnPause: true,
  solid: false,
  textVariant: null,
  title: undefined,
  toastClass: undefined,
  transProps: undefined,
  // Link props
  // All others use defaults
  noRel: undefined,
  active: undefined,
  activeClass: undefined,
  disabled: undefined,
  exactActiveClass: undefined,
  href: undefined,
  icon: undefined,
  opacity: undefined,
  opacityHover: undefined,
  stretched: false,
  rel: undefined,
  replace: undefined,
  routerComponentName: undefined,
  target: undefined,
  to: undefined,
  underlineOffset: undefined,
  underlineOffsetHover: undefined,
  underlineOpacity: undefined,
  underlineOpacityHover: undefined,
  underlineVariant: undefined,
  variant: undefined,
  // End link props
})
const props = useDefaults(_props, 'BToast')

const emit = defineEmits<{
  'close': [value: BvTriggerableEvent]
  'close-countdown': [value: number]
  'hide': [value: BvTriggerableEvent]
  'hidden': [value: BvTriggerableEvent]
  'show': [value: BvTriggerableEvent]
  'shown': [value: BvTriggerableEvent]
  'show-prevented': []
  'hide-prevented': []
}>()

const element = ref<HTMLElement | null>(null)

const modelValue = defineModel<Exclude<BToastProps['modelValue'], undefined>>({default: false})

const {computedLink, computedLinkProps} = useBLinkHelper(props)

// TODO solid is never used
const countdownLength = computed(() =>
  typeof modelValue.value === 'boolean' ? 0 : modelValue.value
)

const {
  isActive,
  pause,
  restart,
  resume,
  stop,
  isPaused,
  value: remainingMs,
} = useCountdown(countdownLength, props.interval, {
  immediate: typeof modelValue.value === 'number',
})
useCountdownHover(
  element,
  computed(() => ({
    noHoverPause: props.noHoverPause,
    noResumeOnHoverLeave: props.noResumeOnHoverLeave,
    modelValueIgnoresHover: typeof modelValue.value === 'boolean',
  })),
  {pause, resume}
)

watchEffect(() => {
  emit('close-countdown', remainingMs.value)
})

const computedTag = computed(() => (computedLink.value ? BLink : 'div'))

const isToastVisible = computed(() =>
  typeof modelValue.value === 'boolean'
    ? modelValue.value
    : isActive.value || (props.showOnPause && isPaused.value)
)

const colorClasses = useColorVariantClasses(props)
const computedClasses = computed(() => [
  colorClasses.value,
  {
    show: isToastVisible.value,
  },
])

const buildTriggerableEvent = (
  type: string,
  opts: Readonly<Partial<BvTriggerableEvent>> = {}
): BvTriggerableEvent =>
  new BvTriggerableEvent(type, {
    cancelable: false,
    target: element.value || null,
    relatedTarget: null,
    trigger: null,
    ...opts,
    componentId: props.id,
  })

const showFn = () => {
  const event = buildTriggerableEvent('show', {cancelable: true})
  emit('show', event)
  if (event.defaultPrevented) {
    if (modelValue.value) modelValue.value = false
    emit('show-prevented')
    return
  }
  if (!modelValue.value) modelValue.value = true
}
const hideFn = (trigger = '') => {
  const event = buildTriggerableEvent('hide', {cancelable: trigger !== '', trigger})

  emit('hide', event)

  if (trigger === 'close') {
    emit('close', event)
  }

  if (event.defaultPrevented) {
    emit('hide-prevented')
    if (!modelValue.value) modelValue.value = true
    return
  }

  if (typeof modelValue.value === 'boolean') {
    modelValue.value = false
  } else {
    modelValue.value = 0
    stop()
  }
}

const onBeforeEnter = () => {
  showFn()
}
const onAfterEnter = () => {
  emit('shown', buildTriggerableEvent('shown'))
}
const onAfterLeave = () => {
  emit('hidden', buildTriggerableEvent('hidden'))
}

// isActive in the composable will cause the toast to hide when the countdown is done
watch(isActive, (newValue) => {
  if (newValue === false && isPaused.value === false && !!modelValue.value) {
    hideFn()
  }
})

defineExpose({
  pause,
  restart,
  resume,
  stop,
})
</script>
