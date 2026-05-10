# bode-sketch 折线 Bode 算法交接说明

## 1. 背景

当前项目是单文件实现，主要代码在 `demo.html`。近期问题集中在“折线草绘模式”的 Bode 图算法，尤其是多个复共轭零/极点的理论转折频率接近时，完整灰色折线响应会出现错接、跳变或 Q peaking/dip 位置不正确。

这份文档只说明折线 Bode 算法相关代码，方便在独立上下文中重构。

## 2. 相关状态与数据

主要状态在 `state`：

- `state.zeros`：零点条目数组。
- `state.poles`：极点条目数组。
- `state.mode`：`exact` 或 `sketch`。
- `state.selected`：当前选中的零/极点条目。
- `state.ranges`：Bode 图频率、幅度、相位范围设置。

每个零/极点条目大致为：

```js
{ id, kind, re, im }
```

其中：

- `im === 0` 表示实零/极点。
- `im > 0` 表示一对共轭复零/极点，实际点为 `re ± j*im`。

## 3. 当前调用链

折线 Bode 图主要调用链：

```text
render()
  -> renderPlots()
    -> currentBode()
      -> exactFreqs = logspace(...)
      -> exact = bodeExact(exactFreqs)
      -> sketchVerts = sketchVertices(...)
      -> sketchFreqs = sketchVerts.map(v => v.w)
      -> active = bodeSketch(sketchFreqs, ..., { vertices: sketchVerts })
    -> selectedContribution(...)
    -> drawBodePlot(...)
```

相关函数位置均在 `demo.html`。

## 4. 当前相关函数

### 精确响应

- `bodeExact(freqs)`
- `bodeExactFor(freqs, zeroEntries, poleEntries, k)`
- `hAt(w, entriesZeros, entriesPoles, k)`
- `gainK()`

这些函数负责精确计算 `H(jω)` 的幅度和相位。

### 折线近似基础函数

- `firstOrderSketchTerm(w, w0, sign, order)`
  - 用于幅度渐近线。
  - `sign = +1` 表示零点贡献，`sign = -1` 表示极点贡献。
  - `order = 1` 表示实零/极点，`order = 2` 表示共轭复零/极点。

- `phaseSketchTerm(w, w0, sign, order, q)`
  - 用于相位折线。
  - 当前二阶过渡范围使用：

```text
ω_low  = ω0 * 10^(-1/(2Q))
ω_high = ω0 * 10^(+1/(2Q))
```

- `sketchBounds(entry)`
  - 返回 `{ lo, w0, hi, q }`。
  - `lo = ω0 * 10^(-1/(2Q))`
  - `hi = ω0 * 10^(+1/(2Q))`
  - 实零/极点默认 `Q = 0.5`。

- `sketchEntryMag(entry, kind, w)`
  - 当前只计算不含 Q peaking/dip 的幅度渐近线。

- `sketchPeakDb(entry, kind)`
  - 当前显式返回复共轭项在 `ω0` 的 peak/dip：
    - 复极点：`+20log10(Q)`
    - 复零点：`-20log10(Q)`

- `sketchEntryPhase(entry, kind, w)`
  - 当前按零/极点类型和左右半平面决定相位方向。

### 折线事件与顶点

- `sketchVertices(zeroEntries, poleEntries, wMin, wMax)`
  - 当前尝试生成结构化顶点。
  - 每个 entry 贡献：
    - `{ w: lo, type: "lo", entry, kind }`
    - `{ w: w0, type: "center", entry, kind }`
    - `{ w: hi, type: "hi", entry, kind }`
  - `wMin/wMax` 作为 endpoint。
  - 当前只合并几乎同频的事件，容差常量为 `SKETCH_EVENT_TOL`。

- `sketchFrequencies(...)`
  - 兼容函数，只返回 `sketchVertices(...).map(v => v.w)`。

### 折线求值

- `bodeSketch(freqs, exact, zeroEntries, poleEntries, k, options)`
  - 当前使用 `freqs` 和可选 `options.vertices`。
  - 先计算不含 peak 的渐近线基线。
  - 若 `includePeaking !== false`，则遍历当前 vertex 的 `events`，对 `type === "center"` 的事件加入 `sketchPeakDb()`。

### 选中对象贡献

- `selectedContribution(freqs, full, fullFreqs, mode, options)`
  - 精确模式下计算选中对象的精确响应。
  - 折线模式下调用 `bodeSketch(...)`。
  - 用 `magOffset/phaseOffset` 把选中对象贡献平移到和完整响应在选中对象 `ω0` 附近对齐。

### 绘图

- `drawBodePlot(svg, width, height, freqs, series, yMin, yMax, label)`
  - 支持普通曲线。
  - 支持 `kind: "vline"` 的垂直辅助线。
  - 支持 `kind: "rangeArrow"` 的水平双箭头。

## 5. 相关函数索引

当前 `demo.html` 中重点关注以下函数：

- `firstOrderSketchTerm()`
- `phaseSketchTerm()`
- `sketchBounds()`
- `sketchEntryMag()`
- `sketchPeakDb()`
- `sketchEntryPhase()`
- `sketchVertices()`
- `sketchFrequencies()`
- `bodeSketch()`
- `currentBode()`
- `selectedContribution()`
- `renderPlots()`
- `drawBodePlot()`
