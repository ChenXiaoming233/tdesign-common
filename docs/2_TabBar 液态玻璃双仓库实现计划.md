# TabBar 液态玻璃双仓库实现计划

## Summary

以两个配套 PR 交付：

1. `tdesign-common`：公共 LESS 材质、主题 token、兼容降级和文档入口。
2. `tdesign-mobile-vue`：`effect` API、内部 SVG 渐进增强、测试和可运行 Demo。

采用“可读基底 + CSS 玻璃 fallback + Chromium 边缘位移增强”的三级能力架构。禁止复制、改写或拼接竞争 PR、开源项目中的源码、SVG、参数和素材；仅依据功能目标独立设计实现。无第三方运行时依赖，不使用 WebGL、WebGPU、背景复制或调用方提供的外部 filter id。静态 SVG 是首选增强实现；若原型证明静态 SVG 无法在目标尺寸稳定产生可观察位移，可启用原创 Canvas 2D Plan B，仅生成内部位移/高光图。Canvas、尺寸观察和一次性调度是受控实现预算，不是 TDesign 全局禁令；持续空闲任务、`window`/`document` 全局监听和无限动画仍然排除。

## Public API

- 新增 `effect?: 'normal' | 'glass'`，默认 `normal`。
- 保留 `shape?: 'normal' | 'round'`，不增加 `round-glass`；`shape` 决定轮廓，`effect` 决定材质。
- `glass` 仅在 `shape="round"` 时生效；`theme` 只决定选中项是普通内容态还是 `tag` 内胶囊，不参与玻璃外壳的启用判断。
- `effect="glass" + shape="normal"` 按普通模式渲染，不产生多余 class、SVG 或警告。
- glass 有效组合在 scoped CSS 中隐藏 `.@{item}--split::before` hairline，但不移除 `split` prop 或 `.@{item}--split` class。动态移除 glass 后分割线按原有 `split` 行为恢复；normal 和非 glass round 模式不受影响。`round-glass.vue` 不额外传入 `:split="false"`，用于证明推荐调用只需 `shape="round" effect="glass"`。
- 双仓库内部契约固定为 glass modifier 和私有滤镜变量 `--td-tab-bar-glass-filter`：mobile-vue 生成 `${tabBarClass.value}--glass`，common 使用 `.@{tab-bar-cls}--glass`；默认前缀下的实际 class 是 `t-tab-bar--glass`。common 只在 round 与 glass 两个 modifier 同时存在时启用玻璃样式。私有变量不属于公共定制 API，不得在 `_var.less` 中声明或进入 CSS Variables 文档。
- 不开放 SVG、背景图片、折射强度等组件 props，避免把浏览器实现细节变成公共 API。
- 不采用调用方提供背景副本、滤镜 ID 或背景定位的方案；这会要求业务同步滚动、尺寸、主题和动态背景，降低组件泛用性，并把浏览器实现细节暴露给调用方。
- 当前方案新增以下 9 个 CSS variables；现有 TabBar CSS variables 保持不变：
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
- 公开 CSS variables 原则上保持上述 9 个，不为单个高光、阴影或状态随意拆分变量；如实现证明存在新的、独立且稳定的主题定制职责，可在评审中说明后增加。私有 `--td-tab-bar-glass-filter` 只承担双仓库内部引用。不得删除或重命名现有 TabBar CSS variables。
- normal 模式不增加装饰 DOM；glass 模式优先只增加一个 optics 层和一组内部 SVG defs。若原型证明单层无法在不破坏 fallback、布局和交互的前提下完成背景边缘位移，新增装饰层必须有独立职责和明确证据。
- 不增加 per-item JavaScript，不扫描全局 DOM，不监听全局 pointer/mouse 事件。
- 空闲状态不得存在持续 `requestAnimationFrame`、定时器、动画或永久 `will-change`。允许在挂载或实际尺寸/DPR 失效后使用一次性 rAF/nextTick 合并重建，并在任务完成后释放引用。
- 第一版只服务 TabBar，不提前抽象成跨组件 LiquidGlass 系统。
- 第一版明确排除多形状 SDF 融合或 morphing、adaptive tint/backdrop metrics、通用 renderer 抽象和 WebGL/WebGPU 渲染管线；Canvas 仅允许作为本计划定义的单实例位移图 Plan B，不抽象为跨组件 renderer。
- 自动化测试覆盖行为和代表性组合；完整视觉矩阵使用截图验收，避免 snapshot 数量失控。
- 底部反光、额外色散和更强 active Rim Light 均为后续可选优化，不属于本计划的必选交付。

## Implementation

### 独立实现约束

- 从本计划中的行为规格重新编写代码，不 cherry-pick、不复制粘贴、不逐行改写其他 PR 或开源实现。
- SVG filter graph、数值、渐变和图层结构从零设计，不复用现有项目的 data URI、位移贴图、滤镜节点序列或常量。
- 未明确许可证或许可证不完整的开源项目只能用于观察功能目标、架构原则和视觉现象，不得作为源码、SVG、算法表达、参数或素材来源。
- 静态 SVG 位移场必须优先使用 `objectBoundingBox` 或等效的归一化坐标；即使滤镜区域使用 `userSpaceOnUse`，也不得把静态位移参数绑定到固定 TabBar 宽高。若静态 SVG 原型失败，Canvas Plan B 允许根据实例宽高和 DPR 生成内部位移图，但必须使用明确的像素预算、DPR 上限或降采样策略，并验证 320px、390px、430px 和代表性 DPR。
- Demo 使用项目内原创 CSS 色块、线条和内容背景，不引用竞争方案截图、素材或背景。
- 实现阶段不再以其他 PR 源码为工作底稿；只对照公开 API 目标和验收结果。

### `tdesign-common`

1. **建立 token 与主题基线**
   - 在 `tab-bar/_var.less` 和 `tab-bar-item/_var.less` 中按当前职责声明上述 9 个 CSS variables 对应的 LESS 变量。颜色、阴影和动效引用现有语义 token；blur、saturate、edge width 等材质专用标量使用局部默认值，禁止硬编码前景黑白色。若实现需要新增公开变量，必须先证明现有变量无法表达该独立主题职责。
   - 在 `style/mobile/theme/_components.less` 中分别覆盖 light/dark 的公开变量；暗色提高基底遮蔽能力、降低白色高光并使用现有暗色阴影体系。第一级无 blur fallback 还必须为 light/dark 主题选择器直接输出字面量 `background-color`，不能只设置 CSS variable。
   - 主题静态 fallback 选择器的特异性不得阻断第二级增强：在 blur `@supports` 内使用相同的 light/dark 主题选择器重新应用 `background-color: var(--td-tab-bar-glass-bg-color, <literal-fallback>)`。最终级联必须是“主题字面量可读背景 → 同主题 CSS variable 半透明背景 → 可选 SVG optics”。
   - 保持 token 仅描述可稳定定制的材质属性，不暴露 SVG 内部参数。

2. **实现职责单一的四项渲染职责**
   - “四项”是职责划分，不要求创建四个实体 DOM 层；实体装饰 DOM 仍受 Complexity Guardrails 限制。
   - 首选拓扑中根容器只负责定位、圆角、外部阴影和 stacking context，基础材质层承担 `backdrop-filter`。根容器直接设置 `backdrop-filter` 视为可能建立 Backdrop Root、阻断内部背景采样的高风险候选，但不是未经实测的永久禁令；只有原型证明确实不阻断目标浏览器中的背景像素位移时才可采用。
   - 根容器 `::before` 作为基础材质层，负责半透明填充、`blur` 和 `saturate`；它与 optics 是同级绘制层，不是 optics 的滤镜祖先。
   - 首选实体 optics 层是根容器的直接子节点，负责 ring mask 和 SVG URL filter，仅作用于边缘，中心区域保持清晰。optics 与基础材质层的前后关系不能预先写死：必须由前置 Chromium 原型比较“optics 在基础层上方”和“optics 在基础层下方、基础层降低遮蔽”等候选拓扑，确认采样源和可见度后再固定生产结构；如需第二个实体装饰层，必须说明单层失败原因和新增层的单一职责。
   - 无论原型选择哪种拓扑，TabBarItem 内容都必须位于全部装饰层之上；`::after` 只负责静态顶部受光和边缘描边，不运行流光动画。原型必须证明 optics 没有仅采样已经完成的基础模糊层，且基础层没有完全遮蔽背景像素位移。mobile-vue 通过私有 `--td-tab-bar-glass-filter` 传入 `url(#filter-id)`。
   - 各装饰层自行继承圆角，不对根容器设置 `overflow: hidden`，避免裁剪现有 TabBarItem 二级菜单。
   - optics 必须使用 `position: absolute`、`inset: 0`、`flex: none`，不得作为 flex item 参与 TabBarItem 宽度计算。SVG defs 宿主必须使用 `position: absolute`、`width: 0`、`height: 0`、`flex: none`、`pointer-events: none`，不得改变 TabBar 高度或占位高度；所有装饰伪元素和实体节点均不得捕获指针事件。
   - `theme="tag"` 时，TabBarItem 选中层使用半透明内胶囊，并调整内胶囊不透明度、边缘亮度和品牌色内容；`theme="normal"` 保留现有普通选中内容态，只调整语义文字和图标颜色，不新增内胶囊。未选中项透明，继续使用现有语义文字色。
   - 在 glass 组合选择器内将 `.@{item}--split::before` 设为不可见，避免默认 `split=true` 产生内部竖线；不移除 class、不修改 props，也不影响二级菜单或 glass 之外的分割线行为。
   - 不改变现有 TabBarItem 的 flex 分配、宽高、字号、间距和 DOM 结构，不使用固定 item 宽度或 `!important` 重排布局。

3. **实现选中与按压状态**
   - `theme="tag"` 的选中态提高内胶囊不透明度、边缘亮度和品牌色可见度，形成轻微光学响应；`theme="normal"` 不新增内胶囊和底部指示条。
   - `:active` 仅对内容层执行一次短时小幅缩放；不得逐帧调整 blur、阴影或滤镜。
   - 状态 transition 复用项目现有时长和缓动 token，不增加弹簧物理系统。

4. **实现三级能力降级**
   - 第一级是可读基底：在所有 `@supports` 之外先输出最终编译结果不含 `var()`、不依赖 CSS custom properties 或 `backdrop-filter` 的高不透明度背景。light/dark 主题选择器分别直接声明内部字面量背景，不新增公开 CSS variable；支持 CSS variables 但不支持 blur 的浏览器也必须保留这一声明。
   - 第二级是 CSS 玻璃 fallback：仅在正向 `@supports` 检测到标准 `backdrop-filter: blur(...)` 或 `-webkit-backdrop-filter: blur(...)` 后，才使用与第一级相同的 light/dark 主题选择器重新声明 `--td-tab-bar-glass-bg-color` 背景并启用 blur/saturate，确保其特异性能够覆盖静态 fallback。border、shadow 等不影响可读性的公开变量可以在该条件之外使用。
   - 第三级是 SVG 增强：optics 默认无填充、无滤镜且视觉中性；只有对应的 mask、mask-composite、CSS blur 和 SVG URL filter 值均通过语法级 `@supports` 门槛时，才应用 ring mask 与 `--td-tab-bar-glass-filter`。标准路径使用 `mask-composite: exclude`，WebKit 路径使用 `-webkit-mask-composite: xor`。
   - `@supports` 只能作为结构安全和语法门槛，不能证明浏览器实际采样并移动了 backdrop 像素，也不能据此宣传浏览器支持。不得增加 UA 判断；任何未实现或忽略 SVG filter 的环境都继续显示独立存在的 CSS fallback，增强声明只覆盖经过人工验证的 Chromium 版本。
   - `prefers-reduced-motion: reduce` 下关闭 transform 和 transition；静态材质及可读 fallback 保留。

5. **控制变更范围**
   - 修复当前 TabBar 重复的 safe-area 选择器，但不做其他顺手重构。
   - common 的中英文公共 Demo 索引只新增 `round-glass` 占位，不重排或改写无关内容；API 组合与浏览器降级说明由 mobile-vue 的中英文组件文档和 Demo 承载。`round-glass` 仅是 Demo 文件或文档块标识，公共调用方式始终为 `shape="round" effect="glass"`。

### `tdesign-mobile-vue`

1. **接入 API 与有效组合**
   - 在类型、props validator 和中英文文档中加入 `effect`，默认 `normal`。
   - 计算 `isGlass = effect === 'glass' && shape === 'round'`；`theme` 只控制选中项表现，不控制增强 DOM 生命周期。
   - 只有 `isGlass` 为 true 时才向根节点添加 `${tabBarClass.value}--glass`；common 只匹配 `.@{tab-bar-cls}--round.@{tab-bar-cls}--glass`，不根据单独的 `effect` 值猜测状态。默认前缀下对应 `t-tab-bar--round.t-tab-bar--glass`。
   - normal 模式以及无效组合必须保持现有 DOM、class、样式和事件行为，不渲染 SVG 或 optics 层。

2. **接入内部 SVG 渐进增强**
   - 仅在客户端已挂载且 glass 首次生效时分配实例 id。实施前先核对 `tdesign-mobile-vue` 的 Vue 版本、现有唯一 id 工具和 SSR 约定；本计划只要求 id 在同一文档中唯一、实例生命周期内稳定、CSS-safe，并避免多 Vue app、重复 bundle、HMR 和 hydration 冲突，不预先锁定 `Symbol.for`、全局字符串键或特定计数器算法。不得在挂载前访问 `document`。
   - SSR 与客户端首次渲染均不输出 SVG；挂载后才渲染 `aria-hidden="true"` SVG defs 和同时带有 `aria-hidden="true"`、`pointer-events: none` 的 optics 层。SVG defs 宿主和 optics 均不得参与 flex 布局，避免 hydration id 不一致并排除装饰节点的辅助技术语义。
   - 先从零设计轻量静态 SVG 位移场，以小幅 `feDisplacementMap` 形成边缘扰动；不复制背景、不使用外部素材、不使用开源方案的节点序列和参数。
   - 如果静态 SVG 原型未达到可观察且跨尺寸稳定的位移门槛，允许切换到原创 Canvas 2D Plan B：Canvas 只编码边缘位移向量和可选高光，不读取、截图或复制页面背景；生成的内部 data URL 仅作为本实例 SVG `feImage` 的位移输入，Canvas 不作为可见 DOM 节点常驻。Plan B 不得引入 WebGL/WebGPU、通用 renderer 或第三方运行时依赖，并必须在原型阶段确定最大像素预算、DPR 上限或降采样策略。能力不可用、尺寸为零、预算超限或生成失败时保持 optics 中性并直接使用 CSS fallback，不抛出运行时异常。
   - optics 层通过根节点或自身的私有 CSS variable `--td-tab-bar-glass-filter` 消费内部 `url(#filter-id)`，不以内联 `backdrop-filter` 覆盖 common 的能力降级规则，也不接受调用方传入 filter、图片或背景定位。filter id 必须使用私有、稳定且 CSS-safe 的命名，不要求采用固定的递增序号实现。
   - 每个实例只分配一次 id；effect 或 shape 动态切换后增强 DOM 与引用同步增加或移除，再次启用时复用本实例 id。theme 动态切换只更新选中项样式，glass 外壳保持挂载。

3. **保证 SSR 与零空闲成本**
   - 新增 glass 逻辑在服务端与首次 hydration 不读取 `window`、`navigator` 或 `document`；SVG 静态增强和 Canvas Plan B 均在客户端挂载后启动。Plan B 只有在第三级 mask、CSS blur 和 SVG URL filter 等语法门槛全部通过后才初始化，并且只在增强实际启用时为每个 glass 根节点实例化一个 `ResizeObserver`，优先复用现有 placeholder 测量结果；相同尺寸通知不得重复生成，实际尺寸变化后才重建。语法门槛仅用于控制结构与运行时成本，不作为实际视觉支持声明。
   - 不新增 pointer/mouse/全局 resize listener，不启动持续 rAF、定时器或动画；允许使用一次性 rAF/nextTick 合并挂载和尺寸/DPR 失效后的重建。DPR 变化使用可清理的 resolution media query 或仓库既有等效机制感知。卸载或退出 glass 后必须取消待执行任务、断开 observer 和 DPR 监听、清空 SVG data URL 与 Canvas 引用。
   - 不为每个 TabBarItem 创建运行时状态或装饰节点。

4. **提供可评审 Demo 与生成物**
   - 新增独立 `round-glass.vue`（该名称仅为 Demo 标识，不代表公共 API），使用原创多色块、细线和文字背景展示实际穿透，不复制页面背景，也不引用外部素材。
   - Demo 同时展示 normal 与 glass，允许维护者直接比较；选中切换用于验证 active 光学响应。
   - `type.ts`、`props.ts` 和中英文 API 表是生成式产物，但当前仓库没有本地 props 生成命令；按现有生成格式同步修改三者，不虚构生成命令，并在 PR 中注明这一事实。
   - 执行 `npm run api:css -- tab-bar` 更新 CSS variables 文档，执行 `npm run test:demo` 重新生成 Demo 测试；只提交 TabBar 相关生成结果，不带入无关 snapshot 或格式化变化。
   - 不为动态 filter id 引入全局 snapshot serializer，也不在生产代码中增加测试分支。Demo snapshot 允许记录在独立测试环境中按固定用例顺序产生的确定性 id；多实例唯一性和动态切换只由独立行为测试断言。若 snapshot id 在同一提交的重复执行中不稳定，必须先定位测试隔离或计数器泄漏，不能用宽泛字符串替换掩盖问题。
   - 将 `src/_common` 指向 common PR 的精确提交，并在全新 clone 中验证 submodule 可获取。

## Execution Order

1. **隔离功能分支并冻结行为基线**：先在当前 planning checkout 保存 docs 记录点，再让两个仓库的 feature branch 分别从各自最新 `upstream/develop` 派生。优先使用独立 Git worktree 隔离计划 checkout 与功能分支；若采用其他流程，也必须保证内部计划文档 commit 不进入功能分支祖先。记录现有 normal/round TabBar 的 DOM、class、2–5 项布局和关键 snapshot。
2. **完成 Chromium 图层拓扑与增强原型门槛**：先编写最小实验代码，比较 optics 位于基础材质层上方和下方等候选拓扑，并验证根容器是否建立阻断采样的 Backdrop Root。第一阶段验证归一化静态 SVG；若其无法在目标尺寸稳定产生可观察位移，再验证受控 Canvas 2D Plan B。两阶段均使用细线、网格或高对比原创背景直接证明边缘区域存在随背景内容变化的像素位移，同时检查基础层遮蔽和接缝。覆盖 320px、390px、430px、浅色/暗色、多色背景、滚动场景和 DPR 1/2/3 的代表性组合。只有增强可观察、中心内容清晰、连续圆角稳定、无接缝、无明显尺寸差异且重建成本有界时才能继续；两种增强均失败时回退为 CSS glass，不得以普通高光、渐变或静态色散替代背景位移目标。
3. **完成 common 三级降级**：按原型选定的拓扑实现可读基底、CSS glass fallback、light/dark、active、pressed 和 reduced-motion；无 SVG 时前两级必须独立完整。
4. **完成 mobile API 与渐进增强**：接入 `effect`、有效组合 class、唯一 id、内部 defs 和 optics；按原型结果选择静态 SVG 或 Canvas Plan B；验证 normal 模式没有 DOM 差异，任何增强失败都不影响 CSS fallback。
5. **补齐测试与 Demo**：完成行为测试、响应式矩阵、跨浏览器截图和性能证据。
6. **更新 submodule 并提交双 PR**：先固定 common commit，再更新 mobile-vue；两个 PR 互相链接且分别可审查，均不包含本地计划文档。

## Tests And Acceptance

### 自动化行为测试

- 默认 `effect="normal"` 无 `t-tab-bar--glass`、optics 层和 SVG，DOM snapshot 与基线一致。
- `effect="glass" + shape="round"` 在 `theme="normal"` 和 `theme="tag"` 下均产生 `t-tab-bar--glass`，并在客户端挂载后生成唯一滤镜引用。
- `effect="glass" + shape="normal"` 保持普通模式，不渲染增强 DOM。
- 默认 `split=true` 时 glass 仍保留 split class，但 hairline 在组合选择器下不可见；切换回 non-glass 后 hairline 恢复。另验证显式 `split=false` 不产生额外差异。
- 动态切换 effect、shape 能正确增加和移除增强层；切换 theme 不移除 glass 外壳，只更新选中项样式。
- 同页多个 TabBar、多个 Vue app 根节点的 filter id 不重复，引用均指向本实例 defs。
- id 方案在目标运行时中不抛出异常；如果实现依赖可选平台 API，必须覆盖该 API 不可用时的降级路径。
- Demo snapshot 中允许保留隔离测试环境产生的确定性 filter id；连续两次运行必须稳定。具体 ID 唯一性仅通过独立行为测试验证，不依赖 snapshot 推断。
- 使用 `renderToString` 验证 SSR 不输出增强 DOM，再执行 hydration，断言无 hydration warning，挂载完成后才出现 SVG 与 optics。
- spy rAF、定时器、`ResizeObserver` 以及 `window`/`document` 上的 pointer、mouse、resize 监听，证明空闲挂载与切换不会启动持续任务或新增全局监听；允许且需断言 Plan B 每个实例最多一个 observer、一次性重建调度，且不拦截元素现有的 click listener。
- Plan B 下连续的相同尺寸通知不重复生成位移图；一次事件循环内的多次有效失效合并为一次重建，尺寸或 DPR 实际变化才产生新图。切换到 normal、无效 shape 或卸载后 observer、DPR 监听和待执行任务均被清理。
- 模拟 `ResizeObserver`、Canvas context、`toDataURL` 或所需 SVG filter 能力不可用/失败，组件不抛异常、不保留半初始化 optics，并继续显示 CSS fallback。
- 点击、`v-model`、placeholder、fixed、safe-area、Badge 和原有 TabBarItem 事件不回归。
- 重新生成并审查 TabBar Demo snapshot，只接受新增 Demo 及预期节点差异。
- glass 模式新增装饰节点后，TabBarItem 的 flex 计算、宽高、数量布局、placeholder 高度和 safe-area 高度与基线一致。

### 响应式与内容矩阵

- 使用代表性配对而非所有维度的笛卡尔积：至少验证 320px/5 项、375px/3 项、390px/4 项、430px/2 项。
- 在上述组合中分配覆盖纯图标、图标加文字、Badge、长中文和长英文标签；glass 不额外修复 normal 模式既有的长文本行为。
- 所有组合继续使用现有 flex 分配；不得出现固定 item 宽度导致的溢出、压缩错位或安全区遮挡。
- 验证 fixed/non-fixed、placeholder、safe-area 组合，glass 不改变现有组件高度计算和占位逻辑。

### 浏览器与主题矩阵

- Chromium：至少验证一个 Desktop Chrome stable 版本和一个 Android Chrome 或 Android WebView 环境，并在 PR 中记录具体版本；仅在这些已验证的 Chromium 环境中声明 SVG 背景边缘位移增强。
- Chromium 原型在 320px、390px、430px、浅色/暗色、多色背景、滚动场景和 DPR 1/2/3 的代表性组合下，边缘区域应显示随背景变化的可观察像素位移，连续圆角和位移强度保持稳定，且不产生接缝、错位或中心内容位移；未达到条件时不得以轻量高光、渐变或静态色散替代目标。
- Safari/WebKit、Firefox：支持 `backdrop-filter` 时显示完整 CSS 玻璃 fallback，但不承诺 SVG 背景像素位移；不得出现透明消失、文字模糊或空白 optics 遮挡。
- 不支持 `backdrop-filter` 的模拟环境：只使用不含 `var()` 的高不透明度可读基底，图标和文字仍可读。
- 每个浏览器检查 light/dark、纯色/多色背景、normal/glass、多实例和动态选中。
- 模拟 `prefers-reduced-motion: reduce`，确认 transform/transition 关闭且后台没有动画计算。

### 工程与视觉证据

- common 执行 `pnpm lint`、`pnpm test:unit`；mobile-vue 执行 `npm run test:demo`、TabBar unit test、`npm run test`、`npm run lint`、`npm run build` 和 `npm run site:preview`。
- common PR 实际触发的 required checks 和联动框架 CI 必须通过；无法由贡献者本地执行或控制的检查记录远端 CI 结果，不把未触发的全框架任务写成虚假完成项。
- 使用浏览器性能面板记录空闲 glass 模式：无持续 rAF、无持续重绘、无全局事件监听增长；若启用 Plan B，记录一次尺寸/DPR 失效重建的耗时、像素预算和 observer 数量。
- 验证键盘/点击语义、`role="tablist"` 和 TabBarItem 可访问性不变；装饰节点不可聚焦且不进入辅助技术树。
- 记录 normal/glass 对照、浅色、暗色、Safari/Firefox fallback、320px 五项布局和 reduced-motion 截图，并提供一段短录屏展示选择切换。

## PR Visual Specification

PR 描述必须附带一份简洁的原创视觉规格，不新增仓库内长期维护文档：

- **层级表**：基底、optics、高光、选中项分别负责什么，以及明确不包含哪些动态效果。
- **状态表**：normal、glass、active、pressed、reduced-motion 的视觉差异。
- **能力矩阵**：Chromium SVG enhancement、Safari/Firefox CSS fallback、不支持 backdrop-filter 的可读降级。
- **响应式证据**：320px 五项和常规 390px 四项截图，证明未采用固定胶囊宽度。
- **性能声明**：零运行时依赖、零空闲动画、零 `window`/`document` 全局监听、无 WebGL/WebGPU；若启用 Canvas 2D，仅声明其用于按需生成内部位移/高光图，并列出像素预算、重建触发条件、实例级 observer/DPR 监听和清理策略。
- **原创边界**：说明未复制竞争 PR 或开源项目代码、SVG、参数和素材。

## Delivery

- common feature branch 必须从最新 `upstream/develop` 创建，mobile-vue feature branch 同样从其最新 `upstream/develop` 创建；不得从包含本地计划文档 commit 的分支直接派生。
- 两个功能分支优先使用独立 Git worktree；若不使用 worktree，也必须通过分支祖先和 `git diff` 检查证明未携带 planning checkout 的内部文档与提交。
- `docs/1_TDesign TabBar 液态玻璃任务交接文档.md`、`docs/2_TabBar 液态玻璃双仓库实现计划.md` 及其记录点 commit 仅保留在本地，不进入任一官方 PR。
- 推送前分别检查 `git diff --name-only upstream/develop...HEAD`，确认不存在上述计划文档或其他无关文件。
- 先提交 `tdesign-common` PR，说明公共材质、主题、fallback、reduced-motion 和原创实现边界。
- 再提交 `tdesign-mobile-vue` PR，固定 common commit，并互相链接两个 PR 与 issue #2571。
- PR 描述明确列出浏览器能力矩阵、测试命令、截图和无第三方依赖；不声称 Safari/Firefox 支持 SVG 背景像素位移。
- 不提交 Figma：本方案以完整代码 UI、跨主题截图和可运行 Demo 满足 issue 的 UI 设计要求。

## Definition Of Done

- 两个 PR 均可独立审查并互相引用，mobile-vue 的 submodule commit 可由全新 clone 获取。
- normal 模式 DOM、class、布局、事件和视觉无回归。
- glass 在已验证的 Chromium 环境中呈现原创、可观察且随背景变化的边缘像素位移；支持 blur 但没有已验证 SVG 背景位移的环境显示 CSS 玻璃 fallback；不支持 blur 的环境显示高不透明度可读基底。
- light/dark 的无 blur 字面量 fallback 与 blur 增强级联正确；默认 `split=true` 在 glass 中不显示 hairline，退出 glass 后恢复原行为。
- 2–5 项、320–430px、长文本、Badge、safe-area 和 light/dark 全部通过验收。
- active/pressed 状态清晰但克制，reduced-motion 下无 transform、transition 或后台计算。
- 空闲状态无持续 rAF、定时器、`window`/`document` 全局监听、WebGL/WebGPU、持续重绘和新增运行时依赖；若采用 Canvas Plan B，仅在挂载或实际尺寸/DPR 失效后按需生成内部位移/高光图，每个实例最多一个 observer 和一个可清理的 DPR 监听，任务和位图引用在完成、切换和卸载后清理。
- 代码、SVG、数值、素材和 Demo 均为独立实现，没有直接引用竞争 PR 或开源方案源码。
