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

## Implementation

### 独立实现约束

- 从本计划中的行为规格重新编写代码，不 cherry-pick、不复制粘贴、不逐行改写其他 PR 或开源实现。
- SVG filter graph、数值、渐变和图层结构从零设计，不复用现有项目的 data URI、位移贴图、滤镜节点序列或常量。
- Demo 使用项目内原创 CSS 色块、线条和内容背景，不引用竞争方案截图、素材或背景。
- 实现阶段不再以其他 PR 源码为工作底稿；只对照公开 API 目标和验收结果。

### `tdesign-common`

- 在 TabBar LESS 中建立四层静态材质：半透明基底、背景模糊与饱和、顶部镜面高光、边缘描边与轻微色散阴影。
- 增加独立的边缘 optics 层样式，用 ring mask 将 SVG 折射限制在胶囊边缘；中心内容保持清晰。
- 选中项使用独立的半透明内胶囊、边缘高光和品牌色内容；未选中项透明并继续使用语义文字色，不硬编码黑白前景。
- 使用 `:active` 提供轻微缩放反馈，不启用无限流光、pointer tracking 或持续动画；`prefers-reduced-motion: reduce` 下关闭所有 transition/transform 反馈。
- 基础材质始终有效；不支持 `backdrop-filter` 或 SVG URL filter 时，仅失去模糊/折射，仍保留可读的透明填充、描边和阴影。
- 在 `style/mobile/theme/_components.less` 分别定义 light/dark token，暗色模式降低白色高光、提高基底不透明度并调整阴影。
- 修复当前 TabBar 中重复的 safe-area 选择器，但不做其他顺手重构。
- 中英文公共文档增加 `round-glass` Demo 占位和使用边界，不改写无关章节。

### `tdesign-mobile-vue`

- 在类型、props validator 和文档中加入 `effect`；默认值保证现有 DOM、class 和视觉完全不变。
- 计算 `isGlass = effect === 'glass' && shape === 'round' && theme === 'tag'`，仅此时添加 `t-tab-bar--glass`。
- glass 模式客户端挂载后生成实例唯一 filter id，渲染 `aria-hidden` 的内部 SVG defs 和一个 `pointer-events: none` 的 optics 层。
- 独立设计轻量静态 SVG：程序噪声或原创位移场经过小幅 `feDisplacementMap` 产生边缘扰动；不复制背景、不使用 `feImage` 外部素材、不做逐帧动画。
- SVG URL filter 只作用于 optics 层；根节点的 CSS fallback 独立存在，因此 Safari、Firefox、SSR 和不支持滤镜的 WebView 不受影响。
- 服务端和首次 hydration 不访问 `window`、`navigator`、`ResizeObserver`；SVG 增强在 `onMounted` 后启用，卸载时不留下监听器或任务。
- 多实例各自使用唯一 id，切换 `effect` 或不兼容的 shape/theme 后移除增强 DOM。
- 新增独立 `round-glass.vue` Demo，在原创的多色块和细线背景上展示实际背景穿透；不要求调用方复制背景。
- 运行项目现有文档/CSS variable 生成流程，只提交与 TabBar 有关的生成结果，不带入无关快照或格式化修改。
- mobile-vue 的 `src/_common` 指向 common PR 的精确提交，并在全新 clone 中验证 submodule 可以正常初始化。

## Tests And Acceptance

- 单元测试覆盖：
  - 默认 `effect="normal"` 无 glass class、optics 层和 SVG。
  - 合法组合产生 glass class，并在挂载后生成唯一滤镜引用。
  - `normal/tag`、`round/normal` 等无效组合自动保持普通样式。
  - 动态切换 effect 能正确增加和移除增强层。
  - 同页多个 TabBar 的 filter id 不重复。
  - 点击、`v-model`、placeholder、fixed 和 safe-area 原有行为不回归。
- 重新生成并审查 TabBar Demo snapshot，只接受预期差异。
- 执行 common 的完整 lint/typecheck，以及 mobile-vue 的 TabBar unit test、snapshot、lint、build 和 site build。
- 浏览器验收覆盖 Chromium、Safari/WebKit、Firefox，以及 390×844、430×932 两种移动视口。
- 每个浏览器检查 light/dark、纯色/多色背景、normal/glass、reduced-motion、多实例和 fixed safe-area。
- Chromium 应显示轻微边缘折射；Safari/Firefox 应显示完整静态玻璃 fallback，不允许出现透明消失、文字模糊或空白滤镜层。
- 记录浅色、暗色、fallback 和动态选中四组截图，并提供一段短录屏展示切换；使用浏览器性能面板确认空闲状态无 `requestAnimationFrame`、无持续重绘和无事件监听增长。
- 验证键盘/点击语义、`role="tablist"`、TabBarItem 可访问性不变，装饰节点不可聚焦且不进入辅助技术树。

## Delivery

- 先提交 `tdesign-common` PR，说明公共材质、主题、fallback、reduced-motion 和原创实现边界。
- 再提交 `tdesign-mobile-vue` PR，固定 common commit，并互相链接两个 PR 与 issue #2571。
- PR 描述明确列出浏览器能力矩阵、测试命令、截图和无第三方依赖；不声称 Safari/Firefox 支持真实 SVG 折射。
- 不提交 Figma：本方案以完整代码 UI、跨主题截图和可运行 Demo 满足 issue 的 UI 设计要求。
