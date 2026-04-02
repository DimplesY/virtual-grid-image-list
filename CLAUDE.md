# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A Vue 3 + Vite + TypeScript project implementing a high-performance virtual scrolling image grid list using Konva.js canvas rendering. The `VirtualGridImageList` component renders thousands of images efficiently by only painting visible items to an HTML5 canvas.

## Package Manager

This project uses **pnpm**. Do not use npm or yarn.

## Common Commands

```bash
# Development
pnpm dev              # Start Vite dev server

# Building
pnpm build            # Type-check and build for production
pnpm build-only       # Build without type-checking
pnpm type-check       # Run vue-tsc type checking only
pnpm preview          # Preview production build locally

# Linting and Formatting
pnpm lint             # Run all linters (oxlint + eslint)
pnpm lint:oxlint      # Run oxlint only (faster, Rust-based)
pnpm lint:eslint      # Run eslint only
pnpm format           # Run prettier formatting
```

## Architecture

### Tech Stack
- **Vue 3.5** with Composition API (`<script setup>`) and generic component support
- **Vite 8** for build tooling
- **vue-konva** for Konva.js canvas integration
- **TypeScript 6** with strict type checking

### Project Structure

```
src/
├── App.vue                          # Demo app showcasing the component
├── main.ts                          # Entry point, registers vue-konva
└── components/
    └── virtual-grid-image-list/     # Main reusable component
        ├── index.ts                 # Public exports
        └── src/
            └── virtual-grid-image-list.vue   # Component with generic props
```

### Key Architectural Patterns

1. **Canvas-Based Virtual Scrolling**: The `VirtualGridImageList` component uses Konva.js to render images to an HTML5 canvas instead of DOM elements. This enables smooth scrolling with tens of thousands of items.

2. **Generic Component**: Uses Vue 3's generic component syntax (`<script setup lang="ts" generic="T extends Record<string, any>">`) to support custom data types. Users specify image URL and unique key via `imageKey`/`itemKey` props.

3. **Image Caching**: Images are cached in a `Map<string, HTMLImageElement>` after first load. A reactive `loadedUrls` Set triggers re-renders when images finish loading asynchronously.

4. **Overscan Rendering**: The component renders 2 extra rows above and below the visible viewport (`OVERSCAN = 2`) to prevent flickering during scroll.

5. **Path Alias**: `@` is mapped to `./src/` for cleaner imports.

### Props Interface

```typescript
interface Props<T> {
  items: T[]                          // Data array (generic)
  imageKey?: keyof T | ((item: T) => string)  // Image URL field, default 'url'
  itemKey?: keyof T | ((item: T) => string)   // Unique key field, default 'id'
  width?: number                      // Container width
  height?: number                     // Container height
  cellWidth?: number                  // Cell width
  cellHeight?: number                 // Cell height
  gap?: number                        // Cell spacing
  hasMore?: boolean                   // Has more data to load
}
```

### Important Implementation Details

- The component calculates visible rows based on `scrollTop` and only renders items in the viewport plus overscan
- Wheel events on the canvas trigger scroll position updates via `onWheel`
- The `loadMore` event fires when scrolling within 100px of the bottom (virtual infinite scroll)
- Images are loaded asynchronously with batch concurrency control (max 10 per frame)
- Placeholder rectangles show while images are loading

## Linting and Code Quality

- **Oxlint**: Fast Rust-based linter for correctness checks (runs first)
- **ESLint**: Vue-specific and TypeScript rules via `@vue/eslint-config-typescript`
- **Prettier**: Format on save enabled; uses single quotes, no semicolons, 100 char width
- Both linters auto-fix on run (`--fix` flag)

## TypeScript Configuration

- Uses `vue-tsc` instead of `tsc` for type checking (required for `.vue` files)
- `noUncheckedIndexedAccess` is enabled for extra safety
- TypeScript build info cached at `./node_modules/.tmp/tsconfig.app.tsbuildinfo`