<script setup lang="ts">
import { computed } from 'vue';
import { FontAwesomeIcon } from '@fortawesome/vue-fontawesome';

interface ButtonProps {
  /** 按钮文本 */
  label?: string;
  /** 图标组件 */
  icon?: string;
  /** 图标位置 */
  iconPosition?: 'start' | 'end';
  /** 图标颜色 */
  iconColor?: 'red' | 'green' | 'grey';
  /** 是否填充图标 */
  iconFill?: boolean;
  /** 按钮样式 */
  buttonStyle?: 'regular' | 'action' | 'alert' | 'flush';
  /** 是否禁用 */
  disabled?: boolean;
  /** 按钮类型 */
  type?: 'button' | 'submit' | 'reset';
}

const props = withDefaults(defineProps<ButtonProps>(), {
  label: 'Okay',
  iconPosition: 'start',
  buttonStyle: 'regular',
  disabled: false,
  type: 'button',
});

// 定义事件
const emit = defineEmits<{
  click: [event: MouseEvent]
}>();

// 计算class列表
const classList = computed(() => {
  const classes = ['button'];

  if (props.iconColor) {
    classes.push(`icon-${props.iconColor}`);
  }
  if (props.iconFill) {
    classes.push('icon-fill');
  }
  classes.push(`button-style-${props.buttonStyle}`);
  if (props.disabled) {
    classes.push('disabled');
  }

  return classes.join(' ');
});

// 计算图标位置
const startIcon = computed(() =>
  props.iconPosition === 'start' ? props.icon : null
);

const endIcon = computed(() =>
  props.iconPosition === 'end' ? props.icon : null
);


// 点击处理
const handleClick = (event: MouseEvent) => {
  if (!props.disabled) {
    emit('click', event);
  }
};
</script>

<template>
  <button data-component="Button" :class="classList" :disabled="disabled" :type="type" @click="handleClick">
    <!-- Render icon if provided -->
    <FontAwesomeIcon v-if="icon" :icon="icon" />
    <span v-if="startIcon" class="icon icon-start">
      <component :is="startIcon" />
    </span>
    <span class="label">{{ label }}</span>
    <span v-if="endIcon" class="icon icon-end">
      <component :is="endIcon" />
    </span>
  </button>
</template>

<style lang="scss">
@import './Button.scss';
</style>