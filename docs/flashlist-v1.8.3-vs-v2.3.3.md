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

## 7. 严谨场景分析：同 type 高度不齐 +「JS 精确预高」是否让 V1 更优

### 7.1 体感对齐到版本（按机制）

| 体感 | 对齐版本 | 源码机制 |
|---|---|---|
| 惯性跟手、快滑不堵 JS，但白屏率高 | **V1.8.3** | 滚动期主要靠事先算好的绝对坐标 + 小 `drawDistance` 预挂载；坐标/覆盖算错或挂载跟不上 → blank；**没有**每帧对全部 cell 做同步 `measureLayout` |
| 快滑明显卡 / 短暂停住、白屏很少；极端时长列表可能卡死 | **V2.3.3** | 每个 layout 周期对 engaged `ViewHolder` 同步 `measureLayout`；尺寸一变就 `modifyChildrenLayout` → `setRenderId`；超过 40 次 render 未 commit 会告警并停止更新（`RenderTimeTracker`） |

「卡顿反而少白屏」在机制上成立：V2 卡顿把滚速压下来，窗口来得及测完；V1 滚速不降，窗口跑在渲染前面就空。

---

### 7.2 纠正：`getItemType` 在 V1 里主要不是「按 type 求均值」

#### V1：`getItemType` → RLV **回收池 type**；高度均值是**全局**一个窗口

`FlashList.getLayoutProvider`（`v1.8.3/src/FlashList.tsx`）：

1. `getItemType(item)` → RLV layout type（recycle pool 键）
2. `overrideItemLayout` → 可读 `layout.span` / **`layout.size`**
3. 高度：`overrideItemLayout` 的 `size`，否则 fallback **`averageItemSize`**

`GridLayoutProviderWithProps`（`v1.8.3/src/GridLayoutProviderWithProps.ts`）：

- 只有 **一个** `AverageWindow`（不是 `Map<itemType, …>`）
- `reportItemLayout` 把任意 index 的测高丢进**同一个**窗口
- 种子是 `estimatedItemSize`（默认 100）

因此：`getItemType` 对**回收**极重要；它**并不**在 V1 FlashList 里维护「每 type 平均高度」。同 type 高度方差大时，伤的是 fallback 到全局平均、**没写 `layout.size`** 的项。若对**每一项**写准 `size`，全局平均几乎不参与定位。

#### V2：反而是按 type 的 `MultiTypeAverageWindow`；但**不再读取 size 预测**

`RVLayoutManager`（`v2.3.3/.../LayoutManager.ts`）：

- `MultiTypeAverageWindow(5, 200)`，按 `getItemType` 分桶做未测前 bootstrap
- 公开 `overrideItemLayout` 类型只有 `{ span?: number }`；`v2-changes.md` 写明 **不再读 size estimates**
- `SpanSizeInfo` 虽残留 `size?: number`，源码无任何读取——死字段

**「natural 宽高 → JS 精确高度 → 注入布局引擎」这条一等 API，只在 V1 的 `overrideItemLayout.size` 上存在。**

---

### 7.3 V1 白屏因果链（有 / 无精确 size）

V1 写死 `forceNonDeterministicRendering`。RLV `ViewRenderer` 在该模式下：

- cell **不设** height/width，只绝对定位 left/top
- LayoutManager 用 `size` 或全局平均预先堆叠 `y`
- `onLayout`：仅当实际宽高 ≠ 布局记录时才 `overrideLayout` + 排队 relayout

官方 `estimated-item-size.md`：estimate **偏大** → 快滑时以为少量 item 够填屏 → **blank**；组件太慢也会 blank。

| 条件 | 结果 |
|---|---|
| 只用 `estimatedItemSize` / 全局平均，同 type 图高方差大 | 堆叠 `y` 偏 → engaged 覆盖错 → **白屏** |
| **每项** `overrideItemLayout` 写入与真实渲染一致的 `size`（含 separator） | 初始 `y` 准 → blank 大降；且少触发 `onSizeChanged` relayout |
| size 准但 `renderItem` 很重 | 仍可能 blank（挂载跟不上惯性） |
| size 与真实差几 px | 非确定性 relayout + AutoLayout 挪位 |

**「后端 natural + UI 结构 JS 推算」正好打在 V1 blank 主因上。**

---

### 7.4 V2 卡顿因果链

`RecyclerView.tsx` 每个 `useLayoutEffect`：对全部 engaged ViewHolder 同步 `measureItemLayout` → `modifyChildrenLayout` → 变则 `setRenderId`，否则 commit + MVCP。

- 图片先矮后撑 → 反复 measure/relayout → 卡；连续 40 次未 commit → 停止更新（像卡死）
- 即使业务层把算出的 height 写死在 cell 根上，V2 **仍**每轮同步测量 engaged 集合；只是不再高度风暴
- V2 **无法**把 JS 预高写进 layout manager 替代测量

---

### 7.5 结论：精确 JS 预高后，V1 是否更好？

**在你能把预高做到与真实渲染一致的前提下：是。V1 是这条策略的对口引擎；V2 关掉了 size 注入。**

1. API：V1 有 `size`；V2 文档与类型明确不读 size  
2. 滚动热路径：V1 准 size 后主要是换池 + 渲染；V2 多全量 engaged 同步 measure  
3. blank 主因（预堆叠偏大）可被逐项 `size` 消灭  
4. `getItemType` 两边都要（结构分池）；不要误当成 V1 的「每 type 均值」——V1 均值是全局的；精确路径应逐项写 `size` 绕过它  
5. V2 的 per-type 均值只是无估计时的 bootstrap，替补不了你的 JS 模型  

### 7.6 预高必须满足的条件

1. 每个 index 写 `layout.size`（竖列表=高度）  
2. 算式覆盖：图区比例、模块显隐、padding/border、**separator**  
3. Image 按同一公式占位，禁止 `onLoad` 再改高  
4. `overrideItemLayout` 必须快  
5. `getItemType` 按 10 种**结构**，不按高度档  
6. render 仍慢则加大 `drawDistance` 只能缓解  

### 7.7 决策表（修订）

| 能力 | 选择 |
|---|---|
| 能拿到 natural 尺寸且 JS 高度≈真实 UI | **V1.8.3 + 逐项 `overrideItemLayout.size` + 结构 `getItemType`** |
| 推算经常不准 | V1 白屏 / V2 卡顿都会出现——先修占位 |
| 必须用 V2（MVCP/masonry/无原生等） | cell 根写死算出的 height；接受同步 measure；props memo |

### 7.8 V1 精确预高清单

```ts
overrideItemLayout={(layout, item) => {
  layout.size = computeCardHeight(item, listWidth); // 含图比例 + 显隐 + separator
}}
getItemType={(item) => item.cardType} // 10 种结构
estimatedItemSize={medianOrSlightlySmall} // 仅兜底；目标很少走到
```

用 element inspector 对比 `computeCardHeight`；偏差趋近 0 时，快滑 blank 应显著下降，并保持 V1 跟手惯性。
