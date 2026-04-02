<template>
  <v-stage :config="stageConfig" @wheel="onWheel">
    <v-layer>
      <v-rect :config="bgConfig" />

      <v-group
        v-for="row in visibleRows"
        :key="row.rowIndex"
        :config="{ x: 0, y: row.offsetY }"
      >
        <v-group
          v-for="(item, colIdx) in row.cells"
          :key="getItemKey(item)"
          :config="{ x: cellX(colIdx), y: halfGap, listening: true }"
          @click="emit('itemClick', item)"
        >
          <v-rect v-if="!hasImage(getImageUrl(item))" :config="placeholderConfig" />
          <v-image v-else :config="imageConfig(getImageUrl(item))" />
        </v-group>
      </v-group>
    </v-layer>
  </v-stage>
</template>

<script setup lang="ts" generic="T extends Record<string, any>">
import { computed, nextTick, onMounted, shallowRef, watch } from 'vue'

defineOptions({ name: 'VirtualGridImageList' })

const props = defineProps<{
  items: T[]
  /** 获取图片 URL 的字段名或函数 */
  imageKey?: keyof T | ((item: T) => string)
  /** 获取唯一标识的字段名或函数 */
  itemKey?: keyof T | ((item: T) => string)
  width?: number
  height?: number
  rowHeight?: number
  columns?: number
  cellWidth?: number
  cellHeight?: number
  gap?: number
  hasMore?: boolean
}>()

const emit = defineEmits<{
  loadMore: [done: () => void]
  itemClick: [item: T]
}>()

// ============ 默认值处理 ============

const _imageKey = props.imageKey ?? 'url'
const _itemKey = props.itemKey ?? 'id'
const _width = props.width ?? 100
const _height = props.height ?? 100
const _rowHeight = props.rowHeight ?? 140
const _columns = props.columns ?? 4
const _cellWidth = props.cellWidth ?? 100
const _cellHeight = props.cellHeight ?? 100
const _gap = props.gap ?? 8
const _hasMore = props.hasMore ?? false

// ============ 常量定义 ============

const OVERSCAN = 2
const LOAD_MORE_THRESHOLD_PX = 100

const PLACEHOLDER_CONFIG = {
  fill: '#e8e8e8',
  cornerRadius: 6
}

// ============ Getter 函数 ============

function getItemKey(item: T): string {
  if (typeof _itemKey === 'function') {
    return _itemKey(item)
  }
  return String(item[_itemKey])
}

function getImageUrl(item: T): string {
  if (typeof _imageKey === 'function') {
    return _imageKey(item)
  }
  return String(item[_imageKey])
}

// ============ 状态 ============

const scrollTop = shallowRef(0)
const loadingMore = shallowRef(false)
const imageCache = new Map<string, HTMLImageElement>()
const loadedUrls = shallowRef<Set<string>>(new Set())
const pendingLoads = new Set<string>()

const imageConfigCache = new Map<string, ReturnType<typeof buildImageConfig>>()

// ============ 计算属性 ============

const actualColumns = computed(() =>
  _cellWidth
    ? Math.max(1, Math.floor(_width / _cellWidth))
    : _columns
)

const actualGapX = computed(() =>
  _cellWidth
    ? Math.max(
        0,
        (_width - actualColumns.value * _cellWidth) /
          (actualColumns.value + 1)
      )
    : _gap
)

const actualCellW = computed(
  () =>
    _cellWidth ??
    Math.floor((_width - _gap * (_columns + 1)) / _columns)
)

const actualCellH = computed(
  () => _cellHeight ?? _rowHeight - _gap
)

const actualRowHeight = computed(() =>
  _cellHeight ? _cellHeight + _gap : _rowHeight
)

const halfGap = computed(() => Math.floor(_gap / 2))

const totalHeight = computed(
  () =>
    Math.ceil(props.items.length / actualColumns.value) * actualRowHeight.value
)

const maxScrollTop = computed(() =>
  Math.max(0, totalHeight.value - _height)
)

const stageConfig = computed(() => ({
  width: _width,
  height: _height
}))

const bgConfig = computed(() => ({
  x: 0,
  y: 0,
  width: _width,
  height: _height
}))

const placeholderConfig = computed(() => ({
  x: 0,
  y: 0,
  width: actualCellW.value,
  height: actualCellH.value,
  ...PLACEHOLDER_CONFIG
}))

const visibleRows = shallowRef<
  Array<{ rowIndex: number; offsetY: number; cells: T[] }>
>([])
let lastRowStart = -1
let lastRowEnd = -1

function updateVisibleRows() {
  const rowStart = Math.max(
    0,
    Math.floor(scrollTop.value / actualRowHeight.value) - OVERSCAN
  )
  const rowEnd = Math.min(
    Math.ceil(props.items.length / actualColumns.value) - 1,
    Math.ceil((scrollTop.value + _height) / actualRowHeight.value) +
      OVERSCAN
  )

  if (
    rowStart === lastRowStart &&
    rowEnd === lastRowEnd &&
    visibleRows.value.length > 0
  ) {
    const newRows = visibleRows.value.map((row) => ({
      rowIndex: row.rowIndex,
      offsetY: row.rowIndex * actualRowHeight.value - scrollTop.value,
      cells: props.items.slice(
        row.rowIndex * actualColumns.value,
        row.rowIndex * actualColumns.value + actualColumns.value
      )
    }))
    visibleRows.value = newRows
    return
  }

  lastRowStart = rowStart
  lastRowEnd = rowEnd

  const rows: Array<{ rowIndex: number; offsetY: number; cells: T[] }> = []
  for (let r = rowStart; r <= rowEnd; r++) {
    const startIdx = r * actualColumns.value
    rows.push({
      rowIndex: r,
      offsetY: r * actualRowHeight.value - scrollTop.value,
      cells: props.items.slice(startIdx, startIdx + actualColumns.value)
    })
  }
  visibleRows.value = rows
}

// ============ 方法 ============

function cellX(colIdx: number): number {
  return actualGapX.value + colIdx * (actualCellW.value + actualGapX.value)
}

function clampScroll(value: number): number {
  return Math.max(0, Math.min(value, maxScrollTop.value))
}

function hasImage(url: string): boolean {
  if (imageCache.has(url)) return true
  return loadedUrls.value.has(url)
}

function buildImageConfig(url: string) {
  const img = imageCache.get(url)!
  const scale = Math.max(
    actualCellW.value / img.width,
    actualCellH.value / img.height
  )
  const drawW = img.width * scale
  const drawH = img.height * scale
  const offsetX = (actualCellW.value - drawW) / 2
  const offsetY = (actualCellH.value - drawH) / 2

  return {
    image: img,
    x: offsetX,
    y: offsetY,
    width: drawW,
    height: drawH,
    cropX: offsetX / scale,
    cropY: offsetY / scale,
    cropWidth: actualCellW.value / scale,
    cropHeight: actualCellH.value / scale,
    cornerRadius: PLACEHOLDER_CONFIG.cornerRadius
  }
}

function imageConfig(url: string) {
  const img = imageCache.get(url)
  if (!img) {
    return {
      image: null,
      x: 0,
      y: 0,
      width: actualCellW.value,
      height: actualCellH.value
    }
  }

  const cacheKey = `${url}_${actualCellW.value}_${actualCellH.value}`
  let config = imageConfigCache.get(cacheKey)
  if (!config) {
    config = buildImageConfig(url)
    imageConfigCache.set(cacheKey, config)
  }
  return config
}

function loadImage(url: string) {
  if (imageCache.has(url) || pendingLoads.has(url)) return

  pendingLoads.add(url)

  function doLoad() {
    if (loadedUrls.value.has(url)) {
      pendingLoads.delete(url)
      return
    }

    const img = new window.Image()
    img.crossOrigin = 'anonymous'
    img.onload = () => {
      imageCache.set(url, img)
      const newSet = new Set(loadedUrls.value)
      newSet.add(url)
      loadedUrls.value = newSet
      pendingLoads.delete(url)
      updateVisibleRows()
    }
    img.onerror = () => {
      pendingLoads.delete(url)
    }
    img.src = url
  }

  if (pendingLoads.size <= 10) {
    requestAnimationFrame(doLoad)
  } else {
    setTimeout(doLoad, 16 * Math.ceil((pendingLoads.size - 10) / 10))
  }
}

function checkLoadMore() {
  if (!_hasMore || loadingMore.value) return

  const remainingHeight = totalHeight.value - scrollTop.value - _height
  if (remainingHeight <= LOAD_MORE_THRESHOLD_PX) {
    loadingMore.value = true
    emit('loadMore', () => (loadingMore.value = false))
  }
}

function onWheel(e: KonvaWheelEvent) {
  e.evt.preventDefault()
  scrollTop.value = clampScroll(scrollTop.value + e.evt.deltaY)
  updateVisibleRows()
  checkLoadMore()
}

function prependItems(newItems: T[]) {
  const oldRows = Math.ceil(props.items.length / actualColumns.value)
  const newList = [...newItems, ...props.items]
  nextTick(() => {
    const addedRows = Math.ceil(newList.length / actualColumns.value) - oldRows
    scrollTop.value = clampScroll(
      scrollTop.value + addedRows * actualRowHeight.value
    )
    updateVisibleRows()
  })
  return newList
}

function scrollTo(options: { top?: number; index?: number; smooth?: boolean }) {
  const targetTop =
    options.index !== undefined
      ? Math.floor(options.index / actualColumns.value) * actualRowHeight.value
      : (options.top ?? 0)

  const clampedTop = clampScroll(targetTop)

  if (!options.smooth) {
    scrollTop.value = clampedTop
    updateVisibleRows()
    return
  }

  const startTop = scrollTop.value
  const distance = clampedTop - startTop
  const duration = 300
  const startTime = performance.now()

  function animate(currentTime: number) {
    const elapsed = currentTime - startTime
    const progress = Math.min(elapsed / duration, 1)
    scrollTop.value = startTop + distance * easeOutCubic(progress)
    updateVisibleRows()

    if (progress < 1) requestAnimationFrame(animate)
  }

  requestAnimationFrame(animate)
}

function easeOutCubic(t: number) {
  return 1 - (1 - t) ** 3
}

// ============ 类型定义 ============

interface KonvaWheelEvent {
  evt: WheelEvent
}

// ============ 生命周期 ============

onMounted(() => {
  updateVisibleRows()
})

// ============ Watch ============

watch(
  () => props.items,
  () => {
    loadingMore.value = false
    scrollTop.value = clampScroll(scrollTop.value)
    updateVisibleRows()
    nextTick(() => {
      if (
        Math.ceil(props.items.length / actualColumns.value) *
          actualRowHeight.value <
          _height &&
        _hasMore
      ) {
        checkLoadMore()
      }
    })
  },
  { immediate: true }
)

watch(scrollTop, () => {
  updateVisibleRows()
})

watch(
  visibleRows,
  (rows) => {
    const urlsToLoad: string[] = []
    rows.forEach((row) => {
      row.cells.forEach((item) => {
        const url = getImageUrl(item)
        if (!imageCache.has(url) && !pendingLoads.has(url)) {
          urlsToLoad.push(url)
        }
      })
    })

    const batchSize = 10
    let index = 0

    function processBatch() {
      const batch = urlsToLoad.slice(index, index + batchSize)
      batch.forEach((url) => loadImage(url))
      index += batchSize
      if (index < urlsToLoad.length) {
        requestAnimationFrame(processBatch)
      }
    }

    if (urlsToLoad.length > 0) {
      requestAnimationFrame(processBatch)
    }
  },
  { flush: 'post' }
)

// ============ Expose ============

defineExpose({ prependItems, scrollTo })
</script>