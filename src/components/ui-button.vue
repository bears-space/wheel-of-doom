<script setup lang="ts">
import { cn } from '@/lib/utils'

withDefaults(
  defineProps<{
    variant?: 'default' | 'secondary' | 'outline'
    class?: string
    type?: 'button' | 'submit' | 'reset'
    disabled?: boolean
  }>(),
  {
    variant: 'default',
    type: 'button',
  },
)

const variantClasses = {
  default: 'bg-primary text-white hover:opacity-90',
  secondary: 'bg-muted text-foreground hover:bg-muted/80',
  outline: 'border border-border bg-transparent hover:bg-muted/60',
}
</script>

<template>
  <button
    :type="type"
    :disabled="disabled"
    :class="
      cn(
        'inline-flex h-10 items-center justify-center rounded-md px-4 py-2 text-sm font-medium transition disabled:cursor-not-allowed disabled:opacity-50',
        variantClasses[variant],
        $props.class,
      )
    "
  >
    <slot />
  </button>
</template>
