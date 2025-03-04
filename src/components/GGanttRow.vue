<template>
  <div
    class="g-gantt-row"
    :style="rowStyle"
    @mousedown="onDragStart($event)"
    @dragover.prevent="isHovering = true"
    @dragleave="isHovering = false"
    @mouseup="onDragEnd($event)"
    @mouseover="isHovering = true"
    @mouseleave="isHovering = false"
    @drop="onDrop($event)"
  >
    <div
      v-if="label && !labelColumnTitle"
      class="g-gantt-row-label"
      :style="{ background: label.color || colors.primary, color: colors.text }"
    >
      <slot name="label">
        {{ label.name }}
      </slot>
    </div>

    <div
      v-for="(unit, index) in weekendUnits"
      :key="unit.value"
      class="g-gantt-weekend"
      :style="{ left: calculateLeft(unit.date), width: calculateWidth(index) }"
    ></div>

    <!-- **Selection Effect** -->
    <div v-if="isDragging" class="selection-highlight" :style="selectionStyle"></div>

    <div ref="barContainer" class="g-gantt-row-bars-container" v-bind="$attrs">
      <transition-group name="bar-transition" tag="div">
        <g-gantt-bar
          v-for="bar in bars"
          :key="bar.ganttBarConfig.id"
          :bar="bar"
          :highlight="bar.bookingId === highlightedBar"
        >
          <slot name="bar-label" :bar="bar" :highlight="bar.bookingId === highlightedBar" />
        </g-gantt-bar>
      </transition-group>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, type Ref, computed, type StyleValue, provide } from "vue"
import useTimePositionMapping from "../composables/useTimePositionMapping.js"
import provideConfig from "../provider/provideConfig.js"
import type { GanttBarObject } from "../types"
import GGanttBar from "./GGanttBar.vue"
import { BAR_CONTAINER_KEY } from "../provider/symbols"
import dayjs from "dayjs"
import useTimeaxisUnits from "../composables/useTimeaxisUnits.js"

const { timeaxisUnits } = useTimeaxisUnits() // ✅ Get time units

const props = defineProps<{
  label: { name: string; color?: string }
  bars: GanttBarObject[]
  highlightOnHover?: boolean
  highlightedBar: string
}>()

const emit = defineEmits<{
  (e: "dragEvent", value: { start: string | Date; end: string | Date; row: any }): void
  (e: "drop", value: { e: MouseEvent; datetime: string | Date; row: any }): void
  (e: "dragError", value: { message: string; row: any }): void
}>()

const { rowHeight, colors, labelColumnTitle, dateFormat } = provideConfig()
const isHovering = ref(false)
const isDragging = ref(false)
const dragStartTime = ref<string | Date | null>(null)
const dragStartX = ref<number | null>(null)
const dragCurrentX = ref<number | null>(null)
const { mapPositionToTime, mapTimeToPosition } = useTimePositionMapping()
const barContainer: Ref<HTMLElement | null> = ref(null)

provide(BAR_CONTAINER_KEY, barContainer)

const isWeekend = (date: any) => {
  const day = dayjs(date).day()
  return day === 0 || day === 6
}

const weekendUnits = computed(() => {
  return timeaxisUnits.value.lowerUnits.filter((unit) => isWeekend(unit.date))
})

const calculateLeft = (date: any) => {
  return `${mapTimeToPosition(date)}px`
}

const calculateWidth = (unitIndex: number) => {
  const nextUnit = timeaxisUnits.value.lowerUnits[unitIndex + 1]
  const currentUnit = timeaxisUnits.value.lowerUnits[unitIndex]

  if (!nextUnit || !currentUnit) return "50px"

  const format = dateFormat.value || "YYYY-MM-DD HH:mm"
  const nextPosition = mapTimeToPosition(dayjs(nextUnit.date).format(format))
  const currentPosition = mapTimeToPosition(dayjs(currentUnit.date).format(format))

  if (isNaN(nextPosition) || isNaN(currentPosition)) {
    console.error("Invalid positions calculated!")
    return "50px"
  }

  return `${nextPosition - currentPosition}px`
}

const rowStyle = computed(() => {
  return {
    height: `${rowHeight.value}px`,
    background: props.highlightOnHover && isHovering.value ? colors.value.hoverHighlight : null,
    cursor: "pointer"
  } as StyleValue
})

// **Check if selection overlaps with an existing bar**
const isOverlapping = (start: string | Date, end: string | Date) => {
  return props.bars.some((bar) => {
    const barStart = dayjs(bar.myBeginDate)
    const barEnd = dayjs(bar.myEndDate)
    const dragStart = dayjs(start)
    const dragEnd = dayjs(end)

    return (
      (dragStart.isBefore(barEnd) && dragEnd.isAfter(barStart)) || // Dragging inside existing bar
      dragStart.isSame(barStart) ||
      dragEnd.isSame(barEnd) // Exact match
    )
  })
}

// **Compute Selection Box Style**
const selectionStyle = computed<StyleValue>(() => {
  if (!isDragging.value || dragStartX.value === null || dragCurrentX.value === null) {
    return { display: "none" }
  }

  const startX = Math.min(dragStartX.value, dragCurrentX.value)
  const width = Math.abs(dragStartX.value - dragCurrentX.value)

  return {
    left: `${startX}px`,
    width: `${width}px`,
    height: `${rowHeight.value - 10}px`,
    background: "rgba(66, 184, 131, 0.5)",
    position: "absolute",
    top: "5px",
    borderRadius: "4px",
    border: "1px solid #42b883",
    pointerEvents: "none"
  }
})

// **Handles Drag Start**
const onDragStart = (e: MouseEvent) => {
  if ((e.target as HTMLElement).closest(".g-gantt-bar")) {
    return
  }

  const container = barContainer.value?.getBoundingClientRect()
  if (!container) return

  const xPos = e.clientX - container.left
  dragStartX.value = xPos
  dragStartTime.value = mapPositionToTime(xPos)
  isDragging.value = true

  window.addEventListener("mousemove", onDragging)
  window.addEventListener("mouseup", onDragEnd, { once: true })
}

// **Handles Dragging (Mouse Move)**
const onDragging = (e: MouseEvent) => {
  const container = barContainer.value?.getBoundingClientRect()
  if (!container) return

  dragCurrentX.value = e.clientX - container.left
}

// **Handles Drag End**
const onDragEnd = (e: MouseEvent) => {
  window.removeEventListener("mousemove", onDragging)

  if (!isDragging.value || !dragStartTime.value) return

  const container = barContainer.value?.getBoundingClientRect()
  if (!container) return

  const xPos = e.clientX - container.left
  const dragEndTime = mapPositionToTime(xPos)

  if (isOverlapping(dragStartTime.value, dragEndTime)) {
    emit("dragError", {
      message: "Cannot create event: Overlapping existing bar.",
      row: props.label
    })
  } else {
    emit("dragEvent", { start: dragStartTime.value, end: dragEndTime, row: props.label })
  }

  isDragging.value = false
  dragStartTime.value = null
  dragStartX.value = null
  dragCurrentX.value = null
}

// **Handles Drop Event (Preserved)**
const onDrop = (e: MouseEvent) => {
  const container = barContainer.value?.getBoundingClientRect()
  if (!container) {
    console.error("Vue-Ganttastic: failed to find bar container element for row.")
    return
  }
  const xPos = e.clientX - container.left
  const datetime = mapPositionToTime(xPos)
  emit("drop", { e, datetime, row: props.label })
}
</script>

<style>
.g-gantt-row {
  width: 100%;
  transition: background 0.4s;
  position: relative;
  cursor: pointer; /* Indicate draggable behavior */
}

.g-gantt-row > .g-gantt-row-bars-container {
  position: relative;
  border-top: 1px solid #eaeaea;
  width: 100%;
  border-bottom: 1px solid #eaeaea;
}

.g-gantt-row-label {
  position: absolute;
  top: 0;
  left: 0px;
  padding: 0px 8px;
  display: flex;
  align-items: center;
  height: 40%;
  min-height: 20px;
  font-size: 0.8em;
  font-weight: bold;
  border-bottom-right-radius: 6px;
  background: #f2f2f2;
  z-index: 3;
  box-shadow: 0px 1px 4px 0px rgba(0, 0, 0, 0.6);
}

.selection-highlight {
  position: absolute;
  background: rgba(66, 184, 131, 0.5);
  border: 1px solid #42b883;
  border-radius: 4px;
  pointer-events: none;
}

.g-gantt-weekend {
  position: absolute;
  top: 0;
  height: 100%;
  background: #1565c010;
}
</style>
