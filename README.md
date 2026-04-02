# virtual-grid-image-list

一个基于 Vue 3 + Konva.js 的高性能虚拟滚动图片网格列表组件。

## 特性

- **Canvas 渲染**：使用 Konva.js 渲染到 HTML5 Canvas，支持数万张图片流畅滚动
- **虚拟滚动**：仅渲染可视区域内的图片，大幅减少 DOM 节点
- **泛型支持**：支持自定义数据类型，通过 `imageKey`/`itemKey` 指定字段
- **图片懒加载**：异步加载图片并缓存，支持分批并发控制
- **无限滚动**：支持 `loadMore` 事件实现无限加载

## 安装

```sh
pnpm install
```

## 使用示例

```vue
<template>
  <VirtualGridImageList
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
</template>

<script setup lang="ts">
import { ref } from 'vue'
import { VirtualGridImageList } from './components/virtual-grid-image-list'

interface MyImageItem {
  id: string
  url: string
}

const imageList = ref<MyImageItem[]>([
  { id: '1', url: 'https://picsum.photos/seed/1/300/300' },
  { id: '2', url: 'https://picsum.photos/seed/2/300/300' },
  // ...
])

const hasMore = ref(true)

const onLoadMore = (done: () => void) => {
  // 加载更多数据
  done()
}

const onItemClick = (item: MyImageItem) => {
  console.log('clicked:', item.id)
}
</script>
```

## Props

| 属性 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `items` | `T[]` | `[]` | 图片数据数组（泛型） |
| `imageKey` | `keyof T \| ((item: T) => string)` | `'url'` | 图片 URL 字段名或 getter 函数 |
| `itemKey` | `keyof T \| ((item: T) => string)` | `'id'` | 唯一标识字段名或 getter 函数 |
| `width` | `number` | `100` | 容器宽度 |
| `height` | `number` | `100` | 容器高度 |
| `cellWidth` | `number` | `100` | 单元格宽度 |
| `cellHeight` | `number` | `100` | 单元格高度 |
| `gap` | `number` | `8` | 单元格间距 |
| `hasMore` | `boolean` | `false` | 是否有更多数据可加载 |

## Events

| 事件 | 参数 | 说明 |
| --- | --- | --- |
| `loadMore` | `(done: () => void)` | 滚动到底部时触发，调用 `done()` 结束加载状态 |
| `itemClick` | `(item: T)` | 点击图片时触发 |

## Expose Methods

| 方法 | 参数 | 说明 |
| --- | --- | --- |
| `scrollTo` | `{ top?: number, index?: number, smooth?: boolean }` | 滚动到指定位置或索引 |
| `prependItems` | `(newItems: T[])` | 在列表顶部插入数据 |

## 开发

```sh
pnpm dev
```

## 构建

```sh
pnpm build
```

## 类型检查

```sh
pnpm type-check
```

## 代码规范

```sh
pnpm lint
```
