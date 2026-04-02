<template>
  <div class="app">
    <VirtualGridImageList
      ref="listRef"
      :items="imageList"
      :width="660"
      :height="520"
      :cell-width="90"
      :cell-height="90"
      :image-key="(item) => item.url"
      :item-key="(item) => item.id"
      :has-more="hasMore"
      @load-more="onLoadMore"
      @item-click="onItemClick"
    />

    <button @click="scrollTo">Scroll to index 100</button>
  </div>
</template>

<script setup lang="ts">
import { ref, useTemplateRef } from 'vue'
import { VirtualGridImageList } from './components/virtual-grid-image-list'

interface MyImageItem {
  id: string
  url: string
  tagText?: string
}

let nextId = 0

const imageList = ref<MyImageItem[]>(
  Array.from({ length: 120 }, () => ({
    id: `${nextId++}`,
    url: `https://picsum.photos/seed/${nextId}/300/300`,
    tagText: `Label ${nextId}`
  }))
)

const hasMore = ref(true)
const listRef = useTemplateRef('listRef')

const onLoadMore = (done: () => void) => {
  imageList.value.push(
    ...Array.from({ length: 20 }, () => ({
      id: `${nextId++}`,
      url: `https://picsum.photos/seed/${nextId}/300/300`,
      tagText: `Label ${nextId}`
    }))
  )
  if (nextId > 300) hasMore.value = false
  done()
}

const scrollTo = () => {
  listRef.value?.scrollTo({ index: 100, smooth: true })
}

const onItemClick = (item: MyImageItem) => console.log('click', item.id)
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}
.app {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background: #99baeb;
}
</style>
