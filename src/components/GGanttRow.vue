<template>
  <div class="g-gantt-row-wrapper">
    <!-- Label Column -->
    <div
      v-if="!isBlank(label) && !labelColumnTitle"
      class="g-gantt-row-label"
      :style="{ background: colors.primary, color: colors.text }"
    >
      <slot name="label">
        {{ label }}
      </slot>
    </div>

    <!-- Row Content -->
    <div
      class="g-gantt-row"
      :style="rowStyle"
      @dragover.prevent="isHovering = true"
      @dragleave="isHovering = false"
      @drop="onDrop($event)"
      @mouseover="isHovering = true"
      @mouseleave="isHovering = false"
    >
      <div ref="barContainer" class="g-gantt-row-bars-container" v-bind="$attrs">
        <transition-group name="bar-transition" tag="div">
          <g-gantt-bar v-for="bar in bars" :key="bar.ganttBarConfig.id" :bar="bar">
            <slot name="bar-label" :bar="bar" />
          </g-gantt-bar>
        </transition-group>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, type Ref, toRefs, computed, type StyleValue, provide } from "vue"

import useTimePositionMapping from "../composables/useTimePositionMapping.js"
import provideConfig from "../provider/provideConfig.js"
import type { GanttBarObject } from "../types"
import GGanttBar from "./GGanttBar.vue"
import { BAR_CONTAINER_KEY } from "../provider/symbols"

const props = defineProps<{
  label: string
  bars: GanttBarObject[]
  highlightOnHover?: boolean
}>()

const emit = defineEmits<{
  (e: "drop", value: { e: MouseEvent; datetime: string | Date }): void
}>()

const { rowHeight, colors, labelColumnTitle } = provideConfig()
const { highlightOnHover } = toRefs(props)
const isHovering = ref(false)

const rowStyle = computed(() => {
  return {
    height: `${rowHeight.value}px`,
    background: highlightOnHover?.value && isHovering.value ? colors.value.hoverHighlight : null
  } as StyleValue
})

const { mapPositionToTime } = useTimePositionMapping()
const barContainer: Ref<HTMLElement | null> = ref(null)

provide(BAR_CONTAINER_KEY, barContainer)

const onDrop = (e: MouseEvent) => {
  const container = barContainer.value?.getBoundingClientRect()
  if (!container) {
    console.error("Vue-Ganttastic: failed to find bar container element for row.")
    return
  }
  const xPos = e.clientX - container.left
  const datetime = mapPositionToTime(xPos)
  emit("drop", { e, datetime })
}

const isBlank = (str: string) => {
  return !str || /^\s*$/.test(str)
}
</script>

<style>
/* Make the label and row sit next to each other */
.g-gantt-row-wrapper {
  display: flex;
  align-items: center;
  width: 100%;
}

/* Label stays on the left */
.g-gantt-row-label {
  flex: 0 0 150px; /* Fixed width */
  padding: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.9em;
  font-weight: bold;
  border-right: 1px solid #ddd;
  background: #f2f2f2;
  box-shadow: 1px 0px 4px rgba(0, 0, 0, 0.2);
}

/* The row takes the remaining space */
.g-gantt-row {
  flex: 1;
  transition: background 0.4s;
  position: relative;
}

.g-gantt-row-bars-container {
  position: relative;
  border-top: 1px solid #eaeaea;
  width: 100%;
  border-bottom: 1px solid #eaeaea;
}

.bar-transition-leave-active,
.bar-transition-enter-active {
  transition: all 0.2s;
}

.bar-transition-enter-from,
.bar-transition-leave-to {
  transform: scale(0.8);
  opacity: 0;
}
</style>
