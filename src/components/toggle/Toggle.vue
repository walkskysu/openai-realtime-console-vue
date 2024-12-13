<template>
  <div data-component="Toggle" @click="toggleValue" :data-enabled="value.toString()">
    <template v-if="labels">
      <div class="label left" ref="leftRef">
        {{ labels[0] }}
      </div>
      <div class="label right" ref="rightRef">
        {{ labels[1] }}
      </div>
    </template>
    <div class="toggle-background" ref="bgRef"></div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, watch, PropType } from 'vue'

interface Props {
  defaultValue?: string | boolean
  values?: string[]
  labels?: string[]
  onChange?: (isEnabled: boolean, value: string) => void
}

export default defineComponent({
  name: 'Toggle',
  props: {
    defaultValue: {
      type: [String, Boolean],
      default: false
    },
    values: {
      type: Array as () => string[],
      default: () => []
    },
    labels: {
      type: Array as () => string[],
      default: () => []
    },
    onChange: {
      type: Function as PropType<(isEnabled: boolean, value: string) => void>,
      default: () => { }
    }
  },
  setup(props: Props) {
    const leftRef = ref<HTMLDivElement | null>(null)
    const rightRef = ref<HTMLDivElement | null>(null)
    const bgRef = ref<HTMLDivElement | null>(null)

    let initialValue = props.defaultValue
    if (typeof initialValue === 'string') {
      initialValue = !!Math.max(0, (props.values || []).indexOf(initialValue))
    }

    const value = ref<boolean>(initialValue as boolean)

    const toggleValue = () => {
      const v = !value.value
      const index = +v
      value.value = v
      props.onChange?.(v, (props.values || [])[index])
    }

    watch(value, () => {
      const leftEl = leftRef.value
      const rightEl = rightRef.value
      const bgEl = bgRef.value

      if (leftEl && rightEl && bgEl) {
        if (value.value) {
          bgEl.style.left = rightEl.offsetLeft + 'px'
          bgEl.style.width = rightEl.offsetWidth + 'px'
        } else {
          bgEl.style.left = ''
          bgEl.style.width = leftEl.offsetWidth + 'px'
        }
      }
    }, { immediate: true })

    return {
      leftRef,
      rightRef,
      bgRef,
      value,
      toggleValue
    }
  }
})
</script>

<style lang="scss" scoped>
// 保持原有的 Toggle.scss 内容
@import './Toggle.scss'
</style>