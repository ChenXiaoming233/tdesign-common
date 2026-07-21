# TabBar 液态玻璃双仓库实现计划

## Summary

以两个配套 PR 交付：

1. `tdesign-common`：公共 LESS 材质、主题 token、兼容降级和文档入口。
2. `tdesign-mobile-vue`：`effect` API、内部 SVG 渐进增强、测试和可运行 Demo。

采用“CSS/LESS 基础材质 + Chromium SVG 折射增强”的架构。禁止复制、改写或拼接竞争 PR、开源项目中的源码、SVG、参数和素材；仅依据功能目标独立设计实现。无第三方运行时依赖，不使用 Canvas、WebGL、背景复制或调用方提供的外部 filter id。

## Public API

- 新增 `effect?: 'normal' | 'glass'`，默认 `normal`。
- 保留 `shape?: 'normal' | 'round'`，不增加 `round-glass`，使形状与材质正交。
- `glass` 仅在 `shape="round"` 且 `theme="tag"` 时生效；其他组合按普通样式渲染，不产生多余 class、SVG 或警告。
- 不开放 SVG、背景图片、折射强度等组件 props，避免把浏览器实现细节变成公共 API。
- 提供以下 CSS variables：
  - `--td-tab-bar-glass-bg-color`
  - `--td-tab-bar-glass-border-color`
  - `--td-tab-bar-glass-highlight-color`
  - `--td-tab-bar-glass-shadow`
  - `--td-tab-bar-glass-blur`
  - `--td-tab-bar-glass-saturate`
  - `--td-tab-bar-glass-edge-width`
  - `--td-tab-bar-glass-active-bg-color`
  - `--td-tab-bar-glass-active-border-color`

## Complexity Guardrails

- 公共 API 只增加 `effect`，不增加折射强度、背景、动画、滤镜 id 等 props。
- CSS variables 保持上述 9 个，不为单个高光、阴影或状态继续拆分公开变量。
- normal 模式不增加装饰 DOM；glass 模式最多增加一个 optics 层和一组内部 SVG defs。
- 不增加 per-item JavaScript，不扫描全局 DOM，不监听全局 pointer/mouse 事件。
- 空闲状态不得存在 `requestAnimationFrame`、定时器、持续动画或永久 `will-change`。
- 第一版只服务 TabBar，不提前抽象成跨组件 LiquidGlass 系统。
- 自动化测试覆盖行为和代表性组合；完整视觉矩阵使用截图验收，避免 snapshot 数量失控。
- 底部反光、额外色散和更强 active Rim Light 均为后续可选优化，不属于本计划的必选交付。

## Implementation

### 独立实现约束

- 从本计划中的行为规格重新编写代码，不 cherry-pick、不复制粘贴、不逐行改写其他 PR 或开源实现。
- SVG filter graph、数值、渐变和图层结构从零设计，不复用现有项目的 data URI、位移贴图、滤镜节点序列或常量。
- Demo 使用项目内原创 CSS 色块、线条和内容背景，不引用竞争方案截图、素材或背景。
- 实现阶段不再以其他 PR 源码为工作底稿；只对照公开 API 目标和验收结果。

### `tdesign-common`

1. **建立 token 与主题基线**
   - 在 `_var.less` 中声明 9 个 CSS variables 对应的 LESS 变量，默认值引用现有语义色、阴影和动效 token，禁止硬编码前景黑白色。
   - 在 `style/mobile/theme/_components.less` 中分别覆盖 light/dark；暗色提高基底遮蔽能力、降低白色高光并使用现有暗色阴影体系。
   - 保持 token 仅描述可稳定定制的材质属性，不暴露 SVG 内部参数。

2. **实现职责单一的四项渲染职责**
   - “四项”是职责划分，不要求创建四个实体 DOM 层；实体装饰 DOM 仍受 Complexity Guardrails 限制。
   - 根容器负责半透明基底、`blur`、`saturate`、圆角与外部阴影。
   - optics 层负责 ring mask 和 SVG URL filter，仅作用于边缘，中心区域保持清晰。
   - 根容器高光伪元素负责静态顶部受光和边缘描边，不运行流光动画。
   - TabBarItem 选中层负责半透明内胶囊和品牌色内容；未选中项透明，继续使用现有语义文字色。
   - 不改变现有 TabBarItem 的 flex 分配、宽高、字号、间距和 DOM 结构，不使用固定 item 宽度或 `!important` 重排布局。

3. **实现选中与按压状态**
   - 选中态同时提高内胶囊不透明度、边缘亮度和品牌色可见度，形成轻微光学响应，但不新增底部指示条。
   - `:active` 仅对内容层执行一次短时小幅缩放；不得逐帧调整 blur、阴影或滤镜。
   - 状态 transition 复用项目现有时长和缓动 token，不增加弹簧物理系统。

4. **实现两级 fallback**
   - CSS 基础材质先独立成立，SVG URL filter 失败时仍保留 blur、填充、描边和阴影。
   - 使用同时检测标准属性和 `-webkit-backdrop-filter` 的 `@supports` 分支；两者均不支持时提高基底不透明度，确保图标和文字对比度，并隐藏 optics 层。
   - `prefers-reduced-motion: reduce` 下关闭 transform 和 transition；静态材质及可读 fallback 保留。

5. **控制变更范围**
   - 修复当前 TabBar 重复的 safe-area 选择器，但不做其他顺手重构。
   - 中英文公共文档增加 `round-glass` Demo 占位、有效 API 组合和浏览器降级说明，不改写无关章节。

### `tdesign-mobile-vue`

1. **接入 API 与有效组合**
   - 在类型、props validator 和中英文文档中加入 `effect`，默认 `normal`。
   - 计算 `isGlass = effect === 'glass' && shape === 'round' && theme === 'tag'`；只有该组合添加 `t-tab-bar--glass`。
   - normal 模式以及无效组合必须保持现有 DOM、class、样式和事件行为，不渲染 SVG 或 optics 层。

2. **接入内部 SVG 渐进增强**
   - 使用模块级递增计数器生成实例 id；仅在客户端已挂载且 glass 首次生效时分配，格式使用 TDesign 前缀和 TabBar 语义，不使用随机数。
   - SSR 与客户端首次渲染均不输出 SVG；挂载后才渲染 `aria-hidden` SVG defs 与 `pointer-events: none` optics 层，避免 hydration id 不一致。
   - 从零设计轻量静态 SVG 位移场，以小幅 `feDisplacementMap` 形成边缘扰动；不复制背景、不使用外部 `feImage` 素材、不使用开源方案的节点序列和参数。
   - optics 层只消费内部 filter id，不接受调用方传入 filter、图片或背景定位。
   - 每个实例只分配一次 id；effect、shape 或 theme 动态切换后增强 DOM 与引用同步增加或移除，再次启用时复用本实例 id。

3. **保证 SSR 与零空闲成本**
   - 服务端与首次 hydration 不访问 `window`、`navigator`、`document`、`ResizeObserver`。
   - 不注册 pointer/mouse/resize listener，不启动 rAF、定时器或持续动画；卸载后没有任务、节点或引用残留。
   - 不为每个 TabBarItem 创建运行时状态或装饰节点。

4. **提供可评审 Demo 与生成物**
   - 新增独立 `round-glass.vue`，使用原创多色块、细线和文字背景展示实际穿透，不复制页面背景，也不引用外部素材。
   - Demo 同时展示 normal 与 glass，允许维护者直接比较；选中切换用于验证 active 光学响应。
   - 运行现有文档和 CSS variable 生成流程，只提交 TabBar 相关生成结果，不带入无关 snapshot 或格式化变化。
   - 将 `src/_common` 指向 common PR 的精确提交，并在全新 clone 中验证 submodule 可获取。

## Execution Order

1. **冻结行为基线**：记录现有 normal/round TabBar 的 DOM、class、2–5 项布局和关键 snapshot，作为零回归对照。
2. **完成 common CSS fallback**：先在无 SVG 的条件下完成 light/dark、active、pressed、reduced-motion 和不支持 backdrop-filter 的表现。
3. **完成 mobile API**：接入 `effect` 和严格有效组合，验证 normal 模式没有 DOM 差异。
4. **加入 SVG 增强**：在 CSS fallback 稳定后加入唯一 id、内部 defs 和边缘 optics 层；任何失败都不得影响基础材质。
5. **补齐测试与 Demo**：完成行为测试、响应式矩阵、跨浏览器截图和性能证据。
6. **更新 submodule 并提交双 PR**：先固定 common commit，再更新 mobile-vue；两个 PR 互相链接且分别可审查。

## Tests And Acceptance

### 自动化行为测试

- 默认 `effect="normal"` 无 glass class、optics 层和 SVG，DOM snapshot 与基线一致。
- 合法组合产生 glass class，并在客户端挂载后生成唯一滤镜引用。
- `shape="normal" + theme="tag"`、`shape="round" + theme="normal"` 等无效组合保持普通模式。
- 动态切换 effect、shape、theme 能正确增加和移除增强层。
- 同页多个 TabBar 的 filter id 不重复，引用均指向本实例 defs。
- spy `requestAnimationFrame`、定时器和全局 `addEventListener`，证明空闲挂载与切换不会启动持续任务或全局监听。
- 点击、`v-model`、placeholder、fixed、safe-area、Badge 和原有 TabBarItem 事件不回归。
- 重新生成并审查 TabBar Demo snapshot，只接受新增 Demo 及预期节点差异。

### 响应式与内容矩阵

- 在 320、375、390、430px 宽度下分别验证 2、3、4、5 个 TabBarItem。
- 覆盖纯图标、图标加文字、Badge、长中文和长英文标签。
- 所有组合继续使用现有 flex 分配；不得出现固定 item 宽度导致的溢出、压缩错位或安全区遮挡。
- 验证 fixed/non-fixed、placeholder、safe-area 组合，glass 不改变现有组件高度计算和占位逻辑。

### 浏览器与主题矩阵

- Chromium：CSS 基础材质与轻微边缘折射同时可见，中心内容清晰。
- Safari/WebKit、Firefox：完整静态玻璃 fallback，不出现透明消失、文字模糊或空白 optics 层。
- 不支持 `backdrop-filter` 的模拟环境：使用高不透明度基底，图标和文字仍可读。
- 每个浏览器检查 light/dark、纯色/多色背景、normal/glass、多实例和动态选中。
- 模拟 `prefers-reduced-motion: reduce`，确认 transform/transition 关闭且后台没有动画计算。

### 工程与视觉证据

- 执行 common 的完整 lint/typecheck，以及 mobile-vue 的 TabBar unit test、snapshot、lint、build 和 site build。
- 使用浏览器性能面板记录空闲 glass 模式：无 rAF、无持续重绘、无事件监听增长。
- 验证键盘/点击语义、`role="tablist"` 和 TabBarItem 可访问性不变；装饰节点不可聚焦且不进入辅助技术树。
- 记录 normal/glass 对照、浅色、暗色、Safari/Firefox fallback、320px 五项布局和 reduced-motion 截图，并提供一段短录屏展示选择切换。

## PR Visual Specification

PR 描述必须附带一份简洁的原创视觉规格，不新增仓库内长期维护文档：

- **层级表**：基底、optics、高光、选中项分别负责什么，以及明确不包含哪些动态效果。
- **状态表**：normal、glass、active、pressed、reduced-motion 的视觉差异。
- **能力矩阵**：Chromium SVG enhancement、Safari/Firefox CSS fallback、不支持 backdrop-filter 的可读降级。
- **响应式证据**：320px 五项和常规 390px 四项截图，证明未采用固定胶囊宽度。
- **性能声明**：零运行时依赖、零空闲动画、零全局监听、无 Canvas/WebGL。
- **原创边界**：说明未复制竞争 PR 或开源项目代码、SVG、参数和素材。

## Delivery

- 先提交 `tdesign-common` PR，说明公共材质、主题、fallback、reduced-motion 和原创实现边界。
- 再提交 `tdesign-mobile-vue` PR，固定 common commit，并互相链接两个 PR 与 issue #2571。
- PR 描述明确列出浏览器能力矩阵、测试命令、截图和无第三方依赖；不声称 Safari/Firefox 支持真实 SVG 折射。
- 不提交 Figma：本方案以完整代码 UI、跨主题截图和可运行 Demo 满足 issue 的 UI 设计要求。

## Definition Of Done

- 两个 PR 均可独立审查并互相引用，mobile-vue 的 submodule commit 可由全新 clone 获取。
- normal 模式 DOM、class、布局、事件和视觉无回归。
- glass 在 Chromium 有原创边缘折射，在 Safari/Firefox 和无 backdrop-filter 环境有可读 fallback。
- 2–5 项、320–430px、长文本、Badge、safe-area 和 light/dark 全部通过验收。
- active/pressed 状态清晰但克制，reduced-motion 下无 transform、transition 或后台计算。
- 空闲状态无 rAF、定时器、全局监听、Canvas/WebGL、持续重绘和新增运行时依赖。
- 代码、SVG、数值、素材和 Demo 均为独立实现，没有直接引用竞争 PR 或开源方案源码。
