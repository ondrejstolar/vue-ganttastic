<template>
  <div
    :id="barConfig.id"
    :class="['g-gantt-bar', barConfig.class || '', { highlighted: highlight }]"
    :style="{
      ...barConfig.style,
      position: 'absolute',
      top: `${rowHeight * 0.1 + 5}px`,
      left: `${xStart}px`,
      width: `${xEnd - xStart}px`,
      zIndex: isDragging ? 3 : 2
    }"
    @mousedown="onMouseEvent"
    @click="onMouseEvent"
    @dblclick="onMouseEvent"
    @mouseenter="onMouseEvent"
    @mouseleave="onMouseEvent"
    @contextmenu="onMouseEvent"
  >
    <div class="g-gantt-bar-label">
      <slot :bar="bar">
        <div>
          {{ barConfig.label || "" }}
        </div>
        <div v-if="barConfig.html">
          <div class="avatar2">
            <div class="avatar-image">
              <img :src="barConfig.html?.logo" />
            </div>
          </div>
        </div>
      </slot>
    </div>
    <template v-if="barConfig.hasHandles">
      <div class="g-gantt-bar-handle-left" />
      <div class="g-gantt-bar-handle-right" />
    </template>
  </div>
</template>

<script setup lang="ts">
import { computed, ref, toRefs, watch, onMounted, inject } from "vue"

import useBarDragManagement from "../composables/useBarDragManagement.js"
import useTimePositionMapping from "../composables/useTimePositionMapping.js"
import useBarDragLimit from "../composables/useBarDragLimit.js"
import type { GanttBarObject } from "../types"
import provideEmitBarEvent from "../provider/provideEmitBarEvent.js"
import provideConfig from "../provider/provideConfig.js"
import { BAR_CONTAINER_KEY } from "../provider/symbols"

const props = defineProps<{
  bar: GanttBarObject
  highlight: boolean
}>()
const emitBarEvent = provideEmitBarEvent()
const config = provideConfig()
const { rowHeight } = config

const { bar } = toRefs(props)
const { mapTimeToPosition, mapPositionToTime } = useTimePositionMapping()
const { initDragOfBar, initDragOfBundle } = useBarDragManagement()
const { setDragLimitsOfGanttBar } = useBarDragLimit()

const isDragging = ref(false)

const barConfig = computed(() => bar.value.ganttBarConfig)

function firstMousemoveCallback(e: MouseEvent) {
  barConfig.value.bundle != null ? initDragOfBundle(bar.value, e) : initDragOfBar(bar.value, e)
  isDragging.value = true
}

const prepareForDrag = () => {
  setDragLimitsOfGanttBar(bar.value)
  if (barConfig.value.immobile) {
    return
  }

  window.addEventListener("mousemove", firstMousemoveCallback, {
    once: true
  }) // on first mousemove event
  window.addEventListener(
    "mouseup",
    () => {
      // in case user does not move the mouse after mousedown at all
      window.removeEventListener("mousemove", firstMousemoveCallback)
      isDragging.value = false
    },
    { once: true }
  )
}

const barContainerEl = inject(BAR_CONTAINER_KEY)

const onMouseEvent = (e: MouseEvent) => {
  e.preventDefault()
  if (e.type === "mousedown") {
    prepareForDrag()
  }
  const barContainer = barContainerEl?.value?.getBoundingClientRect()
  if (!barContainer) {
    return
  }
  const datetime = mapPositionToTime(e.clientX - barContainer.left)
  emitBarEvent(e, bar.value, datetime)
}

const { barStart, barEnd, width, chartStart, chartEnd, chartSize } = config

const xStart = ref(0)
const xEnd = ref(0)

onMounted(() => {
  watch(
    [bar, width, chartStart, chartEnd, chartSize.width],
    () => {
      xStart.value = mapTimeToPosition(bar.value[barStart.value])
      xEnd.value = mapTimeToPosition(bar.value[barEnd.value])
    },
    { deep: true, immediate: true }
  )
})
</script>

<style>
.g-gantt-bar {
  display: flex;
  justify-content: center;
  align-items: center;
  background: cadetblue;
  overflow: hidden;
}

.g-gantt-bar-label {
  width: 100%;
  height: 100%;
  font-size: 12px;
  box-sizing: border-box;
  padding: 0 7px 0 7px; /* 14px is the width of the handle */
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.g-gantt-bar-label > * {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.g-gantt-bar-handle-left,
.g-gantt-bar-handle-right {
  position: absolute;
  width: 10px;
  height: 100%;
  background: white;
  opacity: 0.7;
  border-radius: 0px;
  cursor: ew-resize;
  top: 0;
}
.g-gantt-bar-handle-left {
  left: 0;
}
.g-gantt-bar-handle-right {
  right: 0;
}

.g-gantt-bar-label img {
  pointer-events: none;
}

.avatar2 {
  display: flex;
  align-items: center;
  justify-content: center;
}

.avatar-image {
  width: 20px;
  height: 20px;
  overflow: hidden;
}

.avatar-image img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.highlighted {
  animation: pulseHighlight 3s infinite alternate;
  background-color: rgba(13, 71, 161, 0.9) !important;
  border: 2px solid #0d47a1 !important;
  color: white !important;
}

@keyframes pulseHighlight {
  0% {
    box-shadow: 0 0 10px 3px rgba(13, 71, 161, 1);
    background-color: rgba(13, 71, 161, 1);
  }
  50% {
    box-shadow: 0 0 12px 4px rgba(13, 71, 161, 0.8);
    background-color: rgba(13, 71, 161, 0.85);
  }
  100% {
    box-shadow: 0 0 10px 3px rgba(13, 71, 161, 1);
    background-color: rgba(13, 71, 161, 1);
  }
}
</style>
