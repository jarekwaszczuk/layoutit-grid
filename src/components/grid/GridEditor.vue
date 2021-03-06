<template>
  <section
    ref="gridEl"
    :class="{ active: isCurrent, dragging }"
    :style="{
      gridTemplateRows,
      gridTemplateColumns,
      gridGap,
      display: 'grid',
    }"
    class="grid"
    @pointerdown="$refs.selection.cellDown($event)"
  >
    <GridCell
      v-for="(section, i) in gridSections(grid)"
      :key="`section-${i}`"
      :area="area"
      :section="section"
      :grayed="!isActive"
      :focused="isFocused(section)"
    />

    <GridTrack
      v-for="track in gridTracks"
      :key="`track-${track.type}-${track.pos}`"
      :area="area"
      :type="track.type"
      :pos="track.pos"
    />

    <GridLine
      v-for="line in gridLines"
      :ref="
        (el) => {
          if (el) {
            gridLineRefs[line.type][line.pos] = el
          }
        }
      "
      :key="`line-${line.type}-${line.pos}`"
      :area="area"
      :type="line.type"
      :pos="line.pos"
      :gap="computedGap[line.type]"
      @down="handleLineDown"
    />

    <GridIntersection
      v-for="intersection in gridIntersections"
      :key="`intersection-${intersection.row}-${intersection.col}`"
      :area="area"
      :row="intersection.row"
      :col="intersection.col"
      :colgap="computedGap.col"
      :rowgap="computedGap.row"
      @down="handleLineDown"
    />

    <AreaEditor v-for="a in areasToShow" :key="`area-${a.name}`" :area="a" @edit="$refs.selection.editArea(a)" />

    <AreaSelection ref="selection" :area="area" @editstart="(a) => (editingArea = a)" @editend="editingArea = null" />
  </section>
</template>

<script setup="props, { el }">
export { default as GridCell } from './GridCell.vue'
export { default as GridTrack } from './GridTrack.vue'
export { default as GridLine } from './GridLine.vue'
export { default as GridIntersection } from './GridIntersection.vue'
export { default as AreaSelection } from './AreaSelection.vue'
export { default as AreaEditor } from '../area/AreaEditor.vue'

export {
  currentArea,
  setCurrentArea,
  valueUnitToString,
  pause,
  resume,
  dragging,
  currentFocus,
  currentHover,
} from '../../store.js'
import { parseValue, parseUnit, parseValueUnit } from '../../store'
import { useIsCurrentArea, useIsActiveArea } from '../../composables/area.js'

import { ref, computed, watch, toRefs, onBeforeUpdate, nextTick } from 'vue'

export { gridSections } from '../../utils.js'

export default {
  props: {
    area: { type: Object, required: true },
  },
}

export const grid = computed(() => props.area.grid)

export const editingArea = ref(null)

export const areasToShow = computed(() => grid.value.areas.filter((a) => a !== editingArea.value))

export const gridLineRefs = ref({ col: [], row: [] })
onBeforeUpdate(() => {
  gridLineRefs.value = { col: [], row: [] }
})

function gridSizesForView(type) {
  return grid.value[type].sizes
    .map((size) => {
      const unit = parseUnit(size)
      switch (unit) {
        case 'auto':
          return '200px'
        case 'min-content':
          return '100px'
        case 'max-content':
          return '300px'
        default:
          return size
      }
    })
    .join(' ')
}

export const gridTemplateRows = computed(() => gridSizesForView('row'))
export const gridTemplateColumns = computed(() => gridSizesForView('col'))

const { area } = toRefs(props)
export const isCurrent = useIsCurrentArea(area)
export const isActive = useIsActiveArea(area)

export const gridEl = ref(null)
export const computedStyles = ref(null)
export const computedGap = ref({ col: '0px', row: '0px' })
watch(
  grid,
  () => {
    nextTick(() => {
      computedStyles.value = window.getComputedStyle(gridEl.value)
      const colGap = parseValueUnit(grid.value.col.gap)
      const rowGap = parseValueUnit(grid.value.row.gap)
      computedGap.value = {
        col:
          colGap.unit === '%'
            ? (parseValue(computedStyles.value.width) / 100) * colGap.value + 'px'
            : computedStyles.value.columnGap,
        row:
          rowGap.unit === '%'
            ? (parseValue(computedStyles.value.height) / 100) * rowGap.value + 'px'
            : computedStyles.value.rowGap,
      }
    })
  },
  { immediate: true, deep: true, flush: 'post' }
)

function tracksFor(type) {
  return grid.value[type].sizes.map((size, i) => {
    return {
      type,
      pos: i + 1,
    }
  })
}
export const gridTracks = computed(() => {
  return [...tracksFor('row'), ...tracksFor('col')]
})

function linesFor(type) {
  const end = grid.value[type].lineNames.length
  const lines = []
  for (let pos = 1; pos <= end; pos++) {
    lines.push({ type, pos })
  }
  return lines
}
export const gridLines = computed(() => {
  return [...linesFor('row'), ...linesFor('col')]
})

export const gridIntersections = computed(() => {
  const rowEnd = grid.value.row.sizes.length
  const colEnd = grid.value.col.sizes.length
  const intersections = []
  for (let row = 2; row <= rowEnd; row++) {
    for (let col = 2; col <= colEnd; col++) {
      intersections.push({ row, col })
    }
  }
  return intersections
})

function toViewGap(gap) {
  // Defaults to 1px so grid gap doesn't disappear
  // return parseValue(gap) === 0 ? '1px' : gap
  return gap
}

export const gridGap = computed(() => {
  return `${toViewGap(grid.value.row.gap)} ${toViewGap(grid.value.col.gap)}`
})

export function isFocused(section) {
  const c = currentHover.value
  return c && c.on === 'cell' && c.grid === grid.value && c.row === section.row.start && c.col === section.col.start
}

function calcValue(prev, prevComp, delta) {
  const sizeAdd = (prev.value * delta) / prevComp.value

  let value = +(prev.value + sizeAdd).toFixed(1)

  if (value <= 0) {
    value = 0.1
  }

  return { value, unit: prev.unit }
}

function resizableUnit(unit) {
  return !(unit === 'auto' || unit === 'max-content' || unit === 'min-content' || unit === 'minmax')
}

function calcSize(size, computedSize, delta) {
  const value = parseValueUnit(size)
  if (resizableUnit(value.unit)) {
    const computedValue = parseValueUnit(computedSize)
    return valueUnitToString(calcValue(value, computedValue, delta))
  }
  return size
}

function resizeGridSizes(sizes, computedSizes, delta, line) {
  const newSizes = [...sizes],
    leftPos = line - 2,
    rightPos = line - 1
  newSizes[leftPos] = calcSize(sizes[leftPos], computedSizes[leftPos], delta)
  newSizes[rightPos] = calcSize(sizes[rightPos], computedSizes[rightPos], -delta)
  return newSizes
}

function farEnough(a, b, delta = 5) {
  return Math.abs(a.x - b.x) > delta || Math.abs(a.y - b.y) > delta
}

export function handleLineDown(event, { row, col }) {
  event.stopPropagation() // TODO: ...
  event.preventDefault()
  if (document.activeElement) {
    document.activeElement.blur()
  }

  if (dragging.value) {
    return
  }

  setCurrentArea(props.area)

  const initialPos = { x: event.clientX, y: event.clientY }
  const initialTime = new Date().getTime()
  const initialRowSizes = [...grid.value.row.sizes]
  const initialRowComputedSizes = computedStyles.value.gridTemplateRows.split(/\s/g)
  const initialColSizes = [...grid.value.col.sizes]
  const initialColComputedSizes = computedStyles.value.gridTemplateColumns.split(/\s/g)
  const rowsNumber = grid.value.row.sizes.length
  const colsNumber = grid.value.col.sizes.length
  const rowLine = row && row > 1 && row <= rowsNumber ? row : undefined
  const colLine = col && col > 1 && col <= colsNumber ? col : undefined
  const handleMove = (event) => {
    const pos = { x: event.clientX, y: event.clientY }

    if (!dragging.value && (new Date().getTime() - initialTime > 500 || farEnough(initialPos, pos))) {
      if (colLine || rowLine) {
        // Start dragging grid lines
        dragging.value = { grid: grid.value, rowLine, colLine }
        document.body.style.cursor = col && row ? 'move' : col ? 'col-resize' : 'row-resize'
        pause()
      }
    }
    if (dragging.value) {
      if (dragging.value.rowLine !== null) {
        // Drag row line by updating row sizes
        grid.value.row.sizes = resizeGridSizes(
          initialRowSizes,
          initialRowComputedSizes,
          pos.y - initialPos.y,
          dragging.value.rowLine
        )
      }
      if (dragging.value.colLine !== undefined) {
        // Drag col line by updating col sizes
        grid.value.col.sizes = resizeGridSizes(
          initialColSizes,
          initialColComputedSizes,
          pos.x - initialPos.x,
          dragging.value.colLine
        )
      }
    }
  }

  const handleUp = () => {
    if (dragging.value) {
      // Finish dragging grid lines
      dragging.value = null
      document.body.style.cursor = 'default'
      resume(true)
    } else if (new Date().getTime() - initialTime < 500) {
      gridLineRefs.value[col ? 'col' : 'row'][col || row].toggleLineName()
    }

    window.removeEventListener('pointermove', handleMove)
    window.removeEventListener('pointerup', handleUp)
  }
  window.addEventListener('pointermove', handleMove)
  window.addEventListener('pointerup', handleUp)
}
</script>

<style scoped lang="scss">
.grid {
  touch-action: none;
  pointer-events: initial;
  height: 100%;
  position: relative;
  background: #300548;
  background: repeating-linear-gradient(45deg, white, white 9px, #f5f5f5 9px, #f5f5f5 14px);
  overflow: hidden;
  user-select: none;
}
</style>
