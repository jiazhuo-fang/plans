# 按实测锁定的现象：从源码追因果

## 0. 现象锁定（不可对调）

| 版本 | 实测 |
|---|---|
| **V1（v1.8.3）** | **卡顿为主，白屏少** |
| **V2（v2.3.3）** | **滚动跟手、不怎么卡，白屏多** |

下文分两层写：

1. **源码事实**：文件里实际做了什么（可核对）  
2. **与现象相容的解释**：事实如何能推出上述体感（标为推断，不是另起一套体感）

---

## 1. V1 源码事实（卡顿 / 少白屏要解释的对象）

### 1.1 引擎与滚动入口

- `FlashList` 包一层 `ProgressiveListView`（`recyclerlistview`），见 `v1.8.3/src/FlashList.tsx`。
- 写死 `forceNonDeterministicRendering`。
- `drawDistance` 默认经 `PlatformConfig.defaultDrawDistance`（native 上 250），并设置：
  - `maxRenderAhead = 3 * drawDistance`
  - `finalRenderAheadOffset = drawDistance`
  - `renderAheadStep = drawDistance`

### 1.2 滚动时 JS 会做什么

RLV 路径（`recyclerlistview` 4.2.3）：

```
Scroll 事件
  → VirtualRenderer.updateOffset
  → ViewabilityTracker 用 layouts[i].y/height 算 engaged
  → renderStack 变化则 RecyclerListView._renderStackWhenReady
  → setState({ renderStack })   // 触发 React 重渲染
```

`layouts[].height` 初始来自：

```
overrideItemLayout.size  ??  GridLayoutProviderWithProps.averageItemSize
```

而 `averageItemSize` 是 **全局一个** `AverageWindow`，种子是 `estimatedItemSize`（默认 100），见 `GridLayoutProviderWithProps.ts`。  
`getItemType` 进的是 RLV **回收 type**，不是这个平均窗口的分桶键。

### 1.3 非确定性渲染 + 尺寸纠偏（额外 JS 工作）

`forceNonDeterministicRendering` 时，`ViewRenderer`：

- style **不设** height/width，只有 absolute `top`/`left`
- `onLayout` 发现真实宽高 ≠ 布局记录 → `onSizeChanged` → `overrideLayout` → `_queueStateRefresh()` → **再次 setState**

粗估计 + 同 type 真实高度差大 → 大量 cell 会反复走这条「测完再整表 relayout」路径。这是源码里明确存在的 **JS 侧重复工作**。

### 1.4 AutoLayout（原生，只动已挂载的人）

`FlashList.container` 包了 `AutoLayoutView`。Android `AutoLayoutView.dispatchDraw`：

1. `fixLayout()` → 按 index 排序已挂载 `CellContainer`，`clearGapsAndOverlaps` 挪 top/left  
2. 再 `super.dispatchDraw`  
3. 若开了 instrumentation，才用真实 scrollOffset 算 blank 并事件上报  

要点（源码注释也写了）：

- 纠正发生在 **draw**，不是多 mount 新 index  
- **不能**给 masonry 用  
- 性能注释认为 child 数量少，原生侧排序/挪位不是大问题  

因此：AutoLayout 改变的是「已有节点的像素位置」，能减少「挂了但中间镂空」的观感；它不负责「再多挂几个 item」。

### 1.5 与「V1：卡、少白」相容的解释（推断）

把事实拼起来，与实测相容的一条链是：

| 环节 | 源码事实 | 对体感的含义 |
|---|---|---|
| A | 粗 `estimatedItemSize` + 全局平均；未写 `overrideItemLayout.size` | 选人用的身高不准 |
| B | 若等效估计 **偏小**（官方也建议拿不准选小）→ engaged **偏多** | 同时存活的复杂卡片多 → **JS setState/render 重 → 卡** |
| C | 非确定性：几乎每张卡 `onLayout` 都可能触发 relayout setState | 滚动中额外 JS 尖峰 → **更卡** |
| D | AutoLayout 在 draw 时把已挂载 cell 挤紧 | 「挂了但缝很大」的镂空被抹掉 → **白屏观感下降** |
| E | 原生 Scroll 仍在滑，但 JS 帧掉了会表现为顿挫 | 用户描述为 **卡顿**，而不是「滑得很顺但中间空」 |

**对三个假说（仅针对 V1 实测）：**

1. **同 type 高度不一致**：通过全局平均 + 非确定性 relayout，主要变成 **卡顿放大器**（事实路径 C；选人偏差也存在）。  
2. **JS 运行时**：`setState(renderStack)` + 反复 `_queueStateRefresh` + 过绘时更重的 `renderItem` → **卡顿主通道**（事实路径 B/C）。  
3. **原生渲染**：AutoLayout 在 UI 线程挪 view；源码自己认为 child 少、不是主成本。它更像 **少白屏的帮凶**（抹缝），不是卡顿主因。

> 局限：源码不能单独证明你当时的粗估计一定偏小；只能证明「偏小 → 过绘 → 卡、少白」与「AutoLayout 抹缝 → 少白」都是真实代码路径。若估计极大，V1 源码同样允许欠绘白屏——那与你的实测相反，说明你那次更可能落在过绘/抹缝路径，而不是欠绘路径。

---

## 2. V2 源码事实（跟手 / 多白屏要解释的对象）

### 2.1 引擎与滚动入口

- `FlashList` ≡ `RecyclerView`（`v2.3.3/src/FlashList.ts`）。
- **无** `android/` / `ios/` 原生列表模块；无 AutoLayout / CellContainer。
- 内部是 RN `ScrollView`（`removeClippedSubviews={false}`，见 `RecyclerView.tsx`）。
- 硬依赖 New Arch（`index.ts` 里非 new arch 直接 throw）。

### 2.2 滚动时 JS 会做什么

```
ScrollView onScroll（原生滚动已经发生）
  → VelocityTracker.computeVelocity
  → EngagedIndicesTracker.updateScrollOffset
       - 可选投影：offset + velocity * averageRenderTime（Android 开 trackAverageRenderTime）
       - 缓冲 2 * drawDistance，按方向 0.3/0.7 分配
  → layoutManager.getVisibleLayouts（二分 layouts[]）
  → 若 engaged 变了 → setRenderId(+1)   // 只调度 React 更新
  → （本次 onScroll 返回；原生惯性可继续）
```

之后才是另一次 React 提交：

```
render ViewHolderCollection
  → useLayoutEffect：
       对 refHolder 里每个 ViewHolder 同步 measureLayout
       → modifyChildrenLayout
       → 若布局变了再 setRenderId
       → 否则 commitLayout + MVCP
  → 连续 40 次 render 未 commit → warn 并停止再 setRenderId
```

未测量过的 item：`estimateLayout` 用 **按 getItemType 分桶** 的 `MultiTypeAverageWindow`（窗口 5，默认约 200），**不是**公开的 `estimatedItemSize`（已删除）。  
`overrideItemLayout` 公开类型只有 `span`；文档写明不再读 size。

### 2.3 没有「画前抹缝」

V1 的 AutoLayout 在 V2 中不存在。  
cell 是 absolute `left`/`top`（`ViewHolder.tsx`）。坐标错了或人还没挂上，视口就是空的——没有原生第二趟去挤紧。

`ViewHolderCollection` 在首次 `commitLayout` 前 `opacity: 0`（防首帧闪），这是首次绘制策略，不是快滑白屏的主路径，但说明 V2 对「未就绪」更倾向隐藏而不是硬画错位。

### 2.4 与「V2：滑、多白」相容的解释（推断）

| 环节 | 源码事实 | 对体感的含义 |
|---|---|---|
| A | 滚动由原生 ScrollView 推进；`onScroll` 里主要是算 engaged + 可能 `setRenderId` | 手势/惯性不必等 cell 画完 → **跟手、不卡** |
| B | 真正挂上/摆对 cell 要等 React commit +（通常）measure | 快滑时窗口已变、节点未到 → **白屏** |
| C | 无 AutoLayout 抹缝 | 人没到或坐标仍是 type 平均估算时，镂空直接可见 → **白屏更刺眼** |
| D | 同 type 高度差大 → 平均 bootstrap 不准；图片撑开后再 measure → 再 `setRenderId` | 空窗时间变长；极端才走到 40 次停更（那是卡死，不是你说的主诉） |

**对三个假说（仅针对 V2 实测）：**

1. **同 type 高度不一致**：让未测/刚测阶段的 `layouts[]` 更不准，并增加二次 measure；是 **白屏持续时间的放大器**，不是「原生画不动」。  
2. **JS 运行时**：这里的形态是 **补挂载/提交滞后**（相对原生滚速），表现为白，而不是把滚动手势拖成顿挫——与 V1「JS 过重导致卡」是同一类资源问题的不同外观。  
3. **原生渲染**：ScrollView 滑得动，恰恰说明原生滚动不是瓶颈；瓶颈在 **JS 填充虚拟化窗口的速度与正确性**。

---

## 3. 对照（只服务锁定现象）

| | V1 实测：卡、少白 | V2 实测：滑、多白 |
|---|---|---|
| 滚动推进 | RLV 跟踪 offset；频繁 `setState(renderStack)` + 可能 relayout `setState` | 原生 ScrollView 先滑；JS 事后 `setRenderId` |
| 尺寸模型 | 公开 estimate / 全局平均 / 可选 `layout.size`；非确定性再测 | 无公开 estimate；type 平均 bootstrap + 同步 measure |
| 缝隙处理 | **有** AutoLayout 画前挪已挂载 cell | **无** |
| 与「卡」相容的主通道 | JS 过绘 + 反复 relayout setState | （你的主诉不是卡；源码里卡死路径是 measure 风暴 / 40 次停更） |
| 与「白」相容的主通道 | 被 AutoLayout + 过绘抑制 → 少白 | 原生已滑走、cell 未 commit/坐标未稳 → 多白 |
| 高度不一致角色 | 加重 V1 的 JS relayout（卡） | 加重 V2 的空窗/错位（白） |

---

## 4. 直接回答三个「是不是」

结合锁定现象 + 源码：

1. **是不是纯原生渲染瓶颈？**  
   **不是。** 两边空白/卡顿都落在「JS 虚拟化决策与提交」；V1 还有原生 AutoLayout，但它是抹缝/观测，不是「GPU 画不动」。

2. **是不是 JS 引擎开销？**  
   **是，但两边外观不同：**  
   - V1：JS 干得太多/太频繁 → **卡**，白被抑制。  
   - V2：JS 补 cell 相对原生滚速太慢/坐标未稳 → **白**，滚动仍跟手。

3. **是不是同 type 高度不一致？**  
   **是放大器，且两边打在不同症状上：**  
   - V1：经全局平均 + 非确定性 relayout → 助长 **卡**。  
   - V2：经 type 平均 bootstrap + 反复 measure → 助长 **白**。  
   它不是唯一根，却是你这个业务（10 种卡、图高不齐、V1 粗估计）里非常关键的条件。

---

## 5. 源码锚点（便于你自己核对）

**V1**

- `src/FlashList.tsx`：`forceNonDeterministicRendering`、`drawDistance`、`AutoLayoutView` 容器  
- `src/GridLayoutProviderWithProps.ts`：全局 `AverageWindow`、`reportItemLayout`  
- RLV `ViewabilityTracker` / `VirtualRenderer` / `_renderStackWhenReady` → `setState`  
- RLV `ViewRenderer` + `_onViewContainerSizeChange` → relayout  
- `android/.../AutoLayoutView.kt` + `AutoLayoutShadow.kt`：draw 时抹缝、blank 观测  

**V2**

- `src/recyclerview/RecyclerView.tsx`：`onScrollHandler`、`useLayoutEffect` measure 环、`removeClippedSubviews={false}`  
- `src/recyclerview/helpers/EngagedIndicesTracker.ts`：缓冲与速度投影  
- `src/recyclerview/layout-managers/LayoutManager.ts`：`MultiTypeAverageWindow`、`getVisibleLayouts`  
- `src/recyclerview/ViewHolder.tsx`：absolute 定位  
- `src/FlashListProps.ts`：`overrideItemLayout` 仅 `span`  
- `documentation/docs/v2-changes.md`：不再读 size estimates  

---

## 6. 诚实边界

- 源码解释的是 **机制可能性与主通道**，不能代替你在真机上对「当时 estimatedItemSize 偏大还是偏小」「白屏是整窗无 node 还是错位镂空」的打点验证。  
- 若要一锤定音，建议在同一列表上打：engaged 数量、每次 scroll 的 setState/setRenderId 次数、AutoLayout blank 回调（V1）、V2 从 scroll 到 ViewHolder mount 的延迟。  
- 此前把体感版本对调后推导的结论作废；以本文 + 你锁定的 V1/V2 现象为准。
