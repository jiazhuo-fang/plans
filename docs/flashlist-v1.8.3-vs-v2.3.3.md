# FlashList 深度对比：v1.8.3 vs v2.3.3

本地源码缓存（未入库）：

- V1 最新：`/workspace/.cache/flash-list/v1.8.3`（tag `v1.8.3`，commit `d7327ed`）
- V2 目标：`/workspace/.cache/flash-list/v2.3.3`（tag `v2.3.3`，commit `527d767`）
- 上游：https://github.com/Shopify/flash-list

重新克隆：

```bash
mkdir -p .cache/flash-list && cd .cache/flash-list
git clone --depth 1 --branch v1.8.3 https://github.com/Shopify/flash-list.git v1.8.3
git clone --depth 1 --branch v2.3.3 https://github.com/Shopify/flash-list.git v2.3.3
```

---

## 一句话结论

两代 FlashList **对外都像 FlatList**，内部都走「绝对定位 + 视窗外回收 React 树」；但引擎完全不同：

| | **v1.8.3** | **v2.3.3** |
|---|---|---|
| 引擎 | 包装 Flipkart `recyclerlistview@4.2.3` | 自研 `src/recyclerview/`（JS-only） |
| 原生 | `AutoLayoutView` + `CellContainer`（Android/iOS） | **无原生模块** |
| 架构要求 | Old / New Arch 都能跑 | **仅 New Architecture（Fabric）** |
| 尺寸 | 强依赖 `estimatedItemSize` | **不需要 estimate**，`useLayoutEffect` 同步 `measureLayout` |
| Masonry | 独立组件 `MasonryFlashList`（嵌套列表） | `masonry` prop + 一等 `MasonryLayoutManager` |

---

## 1. 架构总览

### V1：薄外壳 + 第三方虚拟列表 + 原生纠偏

```
FlashList (PureComponent)
  └─ StickyContainer (recyclerlistview/sticky)
       └─ ProgressiveListView (RLV)
            └─ ScrollView
                 └─ AutoLayoutView (native)          ← 画前清 gap/overlap
                      └─ CellContainer (native) × N  ← 带 index
                           └─ PureComponentWrapper → renderItem
```

职责切分：

- **RLV**：回收池、视窗、绝对 `left/top`、sticky
- **FlashList JS**：FlatList API、`GridLayoutProviderWithProps`（estimate + 滑动平均）、header/footer/viewability
- **Native**：`AutoLayoutView` 在 `dispatchDraw` 里按 index 排序并挪动 cell，修网格跨列造成的缝/叠；可选 `onBlankArea`

入口证据：`v1.8.3/src/FlashList.tsx` 直接 import `ProgressiveListView` / `DataProvider`；`package.json` 依赖 `recyclerlistview@4.2.3`；存在完整 `android/`、`ios/`。

### V2：自研 RecyclerView，整包 JS

```
FlashList ≡ RecyclerView
  └─ ScrollView (CompatScroller)
       ├─ StickyHeaders (Animated overlay)
       ├─ ScrollAnchor (MVCP 纠偏锚点)
       └─ ViewHolderCollection
            └─ ViewHolder × N（absolute left/top）
                 └─ renderItem

RecyclerViewManager
  ├─ LayoutManager (Linear | Grid | Masonry)
  ├─ EngagedIndicesTracker (drawDistance + 速度投影)
  ├─ RenderStackManager (按 itemType 的 key 池)
  └─ ViewabilityManager
```

- `src/FlashList.ts`：`export { RecyclerView as FlashList }`
- `src/index.ts`：非 New Arch 直接 `throw`
- **无** `android/` / `ios/`；`src/native/` 只剩 `PlatformHelper`（drawDistance、inverted transform 等）
- `yarn.lock` 中无 `recyclerlistview`

---

## 2. 相同点（实践时仍适用的心智模型）

### 2.1 FlatList 形 API

两边都保留：`data` / `renderItem` / `keyExtractor` / `getItemType` / `numColumns` / `horizontal` / `inverted` / headers/footers/empty / viewability / `drawDistance` / `scrollTo*` / `prepareForLayoutAnimationRender` 等。

### 2.2 回收语义一致（最关键）

都不是原生 View 池，而是 **JS 层复用 React key → 同一棵子树换 `item` 再渲染**：

- 离开视窗（engaged window）→ key 进 recycle pool
- 新 index 进窗 → 优先同 `getItemType` 的池里取 key
- 因此 **item 内写 `key={item.id}` 会打断回收**；内部 stateful 子组件要能 reset（V2 提供 `useRecyclingState`）

V2 的池实现：`RenderStackManager`（`recycleKeyPools: Map<itemType, Set<key>>`）。  
V1 的池在 RLV 的 `RecycleItemPool` / `VirtualRenderer`，FlashList 用 `getItemType` 喂布局 type。

### 2.3 绝对定位虚拟化

两边 cell 都是 `position: 'absolute'` + 算出的 `x/y`（及宽高策略不同）。内容高度/宽度由布局管理器累加，塞进 ScrollView content。

### 2.4 小缓冲窗口（相对 FlatList）

默认 `drawDistance ≈ 250`（web 更大）。目标是比 FlatList 默认 `windowSize={21}` 更少挂载、更少空白抖动。

### 2.5 Viewability 自建

都不是 VirtualizedList 那套；都有 `ViewabilityHelper` + `ViewabilityManager`，兼容 FlatList 的 `viewableItems` / `changed` / `minimumViewTime`（默认 250ms）。

### 2.6 异构列表靠 `getItemType`

类型决定回收池隔离 → 异构 cell 性能关键杆。

### 2.7 Benchmark 工具线

`useBenchmark` / `useFlatListBenchmark` / `JSFPSMonitor` / `AutoScrollHelper` 等思路延续（V1 另有 `useBlankAreaTracker`，V2 去掉 blank-area 体系）。

---

## 3. 核心差异（实现层）

### 3.1 尺寸与布局：estimate vs 同步测量

**V1**

1. 必须（实践上）给 `estimatedItemSize`；否则用默认约 100，易 blank/overdraw。
2. `GridLayoutProviderWithProps` 用 estimate 种子 + `AverageWindow`；`onLayout` 后更新平均。
3. RLV `forceNonDeterministicRendering`：先按估计放绝对坐标，真实尺寸后 relayout。
4. **Native AutoLayout** 在 paint 前修 gap/overlap（网格跨 span 尤其需要）。Masonry **明确不支持** AutoLayout。

**V2**

1. **无** `estimatedItemSize` / `estimatedListSize` / `estimatedFirstItemOffset`。
2. 内部仍有按 type 的短窗口平均（bootstrap），但对外不暴露。
3. Fabric 下在 `useLayoutEffect` 里对每个 ViewHolder 调 `measureLayout`（相对自身），**同步拿到**宽高，再 `modifyChildrenLayout`；变了就 `setRenderId`，否则 `commitLayout` + MVCP offset。
4. 首屏 `opacity: 0` 直到第一次 commit，避免未测完闪一下。
5. 不再需要 AutoLayout / CellContainer。

关键代码：`v2.3.3/src/recyclerview/utils/measureLayout.ts`（注释写明 `measure` 不准，用 `measureLayout` workaround）。

### 3.2 原生依赖

| | V1 | V2 |
|---|---|---|
| AutoLayoutView | ✅ 修缝/叠 + blank area | ❌ |
| CellContainer | ✅ 携带 index | ❌（普通 View） |
| codegen / pod / gradle | ✅ `rnflashlist` | ❌ |
| 平台适配 | PlatformHelper + 原生 | 仅 PlatformHelper JS |

V1 Android `AutoLayoutView.dispatchDraw` → `clearGapsAndOverlaps`；同时算 blank area 事件。

### 3.3 视窗 / 预渲染策略

**V1（ProgressiveListView）**

- `finalRenderAheadOffset = drawDistance`
- `maxRenderAhead = 3 * drawDistance`，步进 `renderAheadStep = drawDistance`
- 首项 layout 后渐进加大 render-ahead，再落到 final
- `windowShift` 用 header/`distanceFromWindow` 校正

**V2（EngagedIndicesTracker）**

- 总缓冲 `2 * drawDistance`
- 按滚动方向 **0.3 / 0.7** 前后分配（朝滚动方向多预渲染）
- 可选 **offset projection**：`offset + median(velocity) * averageRenderTime`（Android 跟踪真实 render 耗时，其它约 16ms）
- 投影结果仍保证真实 viewport 不被挤出（防 fling 空白带）
- 可见范围用布局上的二分搜索（`findVisibleIndex`）

### 3.4 Layout Manager

| 能力 | V1 | V2 |
|---|---|---|
| 线性 | RLV GridLayoutProvider（1 列） | `LinearLayoutManager` |
| 网格 | `numColumns` + `overrideItemLayout.span`；同行宜等高 | `GridLayoutManager`；**同行可不等高，短项对齐最高项** |
| Masonry | `MasonryFlashList` = 父 FlashList + 每列嵌套 FlashList + 假 ScrollView | `masonry` → `MasonryLayoutManager`；`optimizeItemArrangement` |
| 水平列表 | 常需额外测量 cell；高度受限 | 可 resize；嵌套协调更好 |

### 3.5 Sticky Header

- **V1**：`recyclerlistview/sticky` 的 `StickyContainer`；横向 sticky 直接 throw。
- **V2**：自研 `StickyHeaders.tsx`（Animated + push 动画）；`stickyHeaderConfig`（offset、backdrop、zIndex、`hideRelatedCell` 等）。

### 3.6 maintainVisibleContentPosition（聊天/顶部插入）

- **V1**：基本没有一等、默认开启的完整方案；聊天常靠 `inverted`。
- **V2**：**默认开启**；RN ScrollView MVCP + 自研 `ScrollAnchor`（零高绝对定位节点 `scrollBy`）；支持 `autoscrollToBottom`、`startRenderingFromBottom` 等，可不 inverted 做聊天。

### 3.7 Prop / API 契约变化

**V1 only（已弃用/删除）**

- `estimatedItemSize`, `estimatedListSize`, `estimatedFirstItemOffset`
- `onBlankArea`, `disableAutoLayout`, `disableHorizontalListHeightMeasurement`
- 独立 `MasonryFlashList` / `getColumnFlex`
- 导出 `CellContainer`；`contentContainerStyle` 几乎只认 padding（+ bg）

**V2 only / 强化**

- `masonry`, `optimizeItemArrangement`
- `maintainVisibleContentPosition`（默认 on）
- `onStartReached` / `onStartReachedThreshold`
- `maxItemsInRecyclePool`, `initialDrawBatchSize`, `initialScrollIndexParams`
- `stickyHeaderConfig`, `onCommitLayoutEffect`, `style`
- hooks：`useLayoutState`, `useRecyclingState`, `useMappingHelper`, `useFlashListContext`
- ref：`getLayout`, `computeVisibleIndices`, `getFirstVisibleIndex` 等更完整
- ref 类型从 class instance → `FlashListRef<T>`

### 3.8 嵌套列表

- **V1 Masonry**：嵌套 FlashList + 父滚动驱动子 `onScroll` 的假 ScrollView。
- **V2**：`RecyclerViewContext` 父子协调（`markChildLayoutAsPending`）；水平子列表可在高度已知后提前 unblock 父级；`LayoutCommitObserver` 观察多列表 commit。

### 3.9 性能调参心智

| 主题 | V1 | V2 |
|---|---|---|
| 首要调参 | **估准 `estimatedItemSize`** | 几乎不用估尺寸；更强调 **props memo**、`getItemType`、回收安全 |
| Blank 观测 | `onBlankArea` / `useBlankAreaTracker` | 无；靠测量精度 + 速度投影 |
| Layout animation | `prepareForLayoutAnimationRender` + `keyExtractor` | 同思路；临时 `disableRecycling` |
| 文档提醒 | New Arch「能跑但未充分优化」 | **只为 New Arch 设计** |

---

## 4. 源码地图（后续细聊用）

### V1.8.3 必读

| 路径 | 看什么 |
|---|---|
| `src/FlashList.tsx` | 主壳、RLV 接线、drawDistance、window correction |
| `src/GridLayoutProviderWithProps.ts` | estimate / average / span |
| `src/MasonryFlashList.tsx` | 嵌套列实现 |
| `src/native/auto-layout/*` + `android/.../AutoLayoutView.kt` | 原生纠偏与 blank area |
| `src/native/cell-container/*` | index 载体 |
| `src/viewability/*` | viewability |
| `node_modules/recyclerlistview`（需 yarn） | 真正的回收与 VirtualRenderer |

### V2.3.3 必读

| 路径 | 看什么 |
|---|---|
| `src/recyclerview/RecyclerView.tsx` | 测量 effect、ScrollView、sticky/MVCP 组装 |
| `src/recyclerview/RecyclerViewManager.ts` | 布局选型、progressive render、对外 facade |
| `src/recyclerview/RenderStackManager.ts` | 回收池核心 |
| `src/recyclerview/helpers/EngagedIndicesTracker.ts` | 视窗 + 速度投影 |
| `src/recyclerview/layout-managers/*` | Linear / Grid / Masonry |
| `src/recyclerview/ViewHolder.tsx` | absolute cell |
| `src/recyclerview/utils/measureLayout.ts` | Fabric 同步测量 |
| `src/recyclerview/components/ScrollAnchor.tsx` | MVCP |
| `src/recyclerview/hooks/useRecyclingState.ts` | 回收时重置 state |

---

## 5. 对「基于 FlashList 做高性能列表实践」的含义

1. **若目标是 New Arch / 新项目**：以 **v2.3.3** 为基线；内化 `recyclerview/`（测量 → layout manager → engaged indices → render stack → ViewHolder），而不是 RLV 文档。
2. **若仍在 Old Arch**：只能 **v1.8.3**；要同时懂 RLV + AutoLayout，且接受 estimate 调参。
3. **两代共通纪律**（写 item 组件时）：
   - 用 `getItemType` 拆池
   - 不要在 cell 根上绑业务 `key`
   - 回收后清副作用 / 用 `useRecyclingState`（V2）
   - 控制 `drawDistance` 与复杂子树成本
4. **V2 新增可玩点**：默认 MVCP（聊天）、一等 masonry、网格不等高、更准的 `scrollToIndex`、嵌套列表协调。
5. **不要假设「FlashList = 原生 RecyclerView」**：两代都是 **JS 虚拟列表 + RN ScrollView**；V1 只多了原生「画前纠偏」，V2 用 Fabric 同步测量取代了那层。

---

## 6. 建议的下一轮细聊切入点

按实践优先级：

1. **回收正确性**：`RenderStackManager` / RLV pool + item 组件契约  
2. **布局引擎**：V2 三套 LayoutManager 与 `overrideItemLayout.span`  
3. **滚动窗口**：`EngagedIndicesTracker` 的方向偏置与 velocity projection  
4. **测量闭环**：`useLayoutEffect` → measure → modify → commit / MVCP  
5. **嵌套与聊天**：Context pending、ScrollAnchor、`startRenderingFromBottom`  
6. **从 V1 迁移**：删 estimate、Masonry API、AutoLayout/CellContainer 相关代码路径  

本地缓存已就绪，后续可以直接点名文件/类继续挖。

---

## 7. 场景选型：10 种卡片 + 同 type 内图片高度不齐

### 现象对齐（实现层）

| 体感 | 更吻合的版本 | 底层原因 |
|---|---|---|
| 快滑流畅，但虚白 / 整屏白 | **V1** | 靠 `estimatedItemSize` + 类型平均高估绝对坐标；同 type 高度方差大时，预渲染窗口算不准 → blank；滚动本身较少被同步测量堵住 |
| 快滑卡顿、少白屏；列表很长时可能卡死/crash | **V2** | `useLayoutEffect` 里同步 `measureLayout`；图片异步撑开高度 → 反复 `modifyChildrenLayout` / 重渲染；空白少，但 JS 布局风暴会卡甚至拖垮 |

`getItemType` **解决不了这个问题**。它只隔离回收池（10 种结构 → 10 个池），**不管高度**。同 type 高度参差是布局/测量问题，不是 type 划分问题。不要按「高度档位」拆 type（池会炸、回收变差）。

### 结论：这类业务更该押 **V2**，但有前提

1. **引擎匹配度**：不规则图片流本质是「高度事前不可用单一 estimate 描述」。V2 就是为去掉 estimate、按真实测量排版而重写的；V1 的 blank 在这种数据分布下是结构性的，不是调一下 `estimatedItemSize` 就能根治。
2. **必须先稳住高度**：接口带上图片宽高或 aspect ratio，cell 用固定比例容器占位，图片加载后**不再改变 cell 高度**。否则：
   - V1 → 白屏
   - V2 → 卡顿 / 布局风暴 / crash  
   两边都不会好。
3. **若暂时拿不到尺寸**：短期体感上 V1「更跟手」、V2「更正确但更卡」；产品上整屏白通常比偶发卡顿更伤，但 V2 的 crash 不可接受。正确顺序是 **先做高度占位，再上 V2**，而不是在「裸图片撑开」状态下二选一硬扛。
4. **Old Arch 只能 V1**：那时用 `overrideItemLayout` 尽可能写入已知 size，并把 `estimatedItemSize` 调到接近中位数，接受残余 blank。

### 实践清单（V2 优先路径）

- `getItemType` → 按 **10 种卡片结构** 分，不按高度分
- 图片区：`aspectRatio` / 预知 height，避免 `onLoad` 改布局
- item 组件 memo；回收用 `useRecyclingState`，禁止 cell 内 `key={item.id}`
- 长列表仍卡：查图片解码/重布局次数，适当加大 `drawDistance`，避免一次插入过大
- 只有 New Arch 可用时才上 V2.3.3
