# TDesign TabBar 液态玻璃任务交接文档

## 1. 任务目标

为 [tdesign-common#2571](https://github.com/Tencent/tdesign-common/issues/2571) 实现 iOS 26 风格的 TabBar 悬浮胶囊液态玻璃效果，并通过两个配套 PR 交付：

1. `Tencent/tdesign-common`：公共 LESS 材质、主题 token、兼容降级、公共文档。
2. `Tencent/tdesign-mobile-vue`：Vue API、内部 SVG 增强、测试、Demo。

目标是结合 #2628 和 #2638 的优点，在 API、主题、兼容性、可访问性、测试和工程完整度上超过现有方案。

强制限制：可以参考方向和思路，但禁止复制、改写、拼接或直接引用任何竞争 PR、开源项目的源码、SVG、位移贴图、参数组合和素材。

## 2. 已确认的项目事实

- `Tencent/tdesign` 是项目总入口；公共样式实际维护在 `Tencent/tdesign-common`。
- issue #2571 要求同时完成 UI 设计和至少一个框架侧组件实现。
- 官方认领期为 2026-07-01 至 2026-07-31，7 月下旬集中评审；方案接近时提交时间更早者优先。
- 两组主要竞争方案都是双 PR：
  - [common#2628](https://github.com/Tencent/tdesign-common/pull/2628) + [mobile-vue#2261](https://github.com/Tencent/tdesign-mobile-vue/pull/2261)
  - [common#2638](https://github.com/Tencent/tdesign-common/pull/2638) + [mobile-vue#2268](https://github.com/Tencent/tdesign-mobile-vue/pull/2268)
- 两个仓库的目标分支均为 `develop`。
- `tdesign-mobile-vue/src/_common` 是指向 `tdesign-common` 的 Git submodule。
- 当前工作目录尚不是 Git 仓库，执行 agent 需要准备两个独立 checkout。
- `tdesign-common` 的 browserslist 包含旧浏览器，因此 fallback 是必须能力，不能只实现 Chromium 效果。

## 3. 竞争方案结论

### #2628 的可取方向

- 液态玻璃参数 token 化。
- 包含中心磨砂、边缘层、高光、色散和按压反馈。
- 考虑 `prefers-reduced-motion`。
- 包含 CSS fallback 和框架侧 Demo。

主要不足：

- 将材质并入 `shape="round-glass"`，形状和视觉效果耦合。
- 高级折射要求调用方复制背景并传入 CSS variables。
- 依赖调用方提供外部 SVG filter id。
- token 数量偏多，部分抽象超出 TabBar 当前需求。
- mobile PR 带入无关 DateTimePicker snapshot 变化。

### #2638 的可取方向

- 使用独立的 `effect="normal | glass"`，API 语义更合理。
- 不改变现有 `shape="round"`。
- 改动较小，普通模式兼容性较好。
- 补充了设计使用建议和独立选中项材质。

主要不足：

- 大量视觉参数硬编码，主题能力弱。
- 暗色模式不完整。
- 存在重复 `backdrop-filter` 声明。
- 没有 reduced-motion、真实折射增强和针对性测试。
- `effect="glass"` class 没有严格限制在有效 shape/theme 组合。
- 文档包含较多无关重写，增加评审噪声。

### 共同不足

- 缺少完整的跨浏览器能力矩阵。
- 缺少多实例、动态切换和 SSR/hydration 验证。
- 没有同时做到“真实折射增强”和“零背景复制”。
- 没有充分证明空闲状态性能及 fallback 可读性。

## 4. 开源调研成果

| 项目 | 可参考方向 | 不应移植部分 |
|---|---|---|
| [simple-liquid-glass](https://github.com/lucaperullo/simple-liquid-glass) | SVG 渐进增强、fallback、reduced-motion、可见性和测试意识 | 原有滤镜代码、位移算法、参数、UA 判断及完整组件 |
| [nikdelvin/liquid-glass](https://github.com/nikdelvin/liquid-glass) | CSS/SVG 分层和尺寸无关的组件思路 | Astro、Tailwind、anime.js 和原 SVG 生成代码 |
| [liquid-glass-react](https://github.com/rdev/liquid-glass-react) | 视觉表现和交互方向 | React 实现、滤镜图、鼠标跟踪和布局逻辑 |
| [liquid-glass-effect](https://github.com/kevinbism/liquid-glass-effect) | 纯 CSS fallback 层次 | 将普通 glassmorphism 当作真实折射 |
| [liquid-glass-vue](https://github.com/WXperia/liquid-glass-vue) | Vue 组件接口拆分方向 | 源码、SSR 不安全实现；仓库缺少完整 LICENSE 文件 |
| [liquid-glass-studio](https://github.com/iyinchao/liquid-glass-studio) | 仅用于视觉观察 | WebGL2/WebGPU，体积和复杂度不适合 TDesign |

浏览器结论：SVG URL filter 用于 backdrop 折射主要视为 Chromium 渐进增强；Safari、iOS、Firefox 必须得到完整 CSS 玻璃 fallback，不能宣传为支持真实折射。

## 5. 已锁定的技术决策

### Public API

- 新增 `effect?: 'normal' | 'glass'`，默认 `normal`。
- 保留 `shape?: 'normal' | 'round'`，不增加 `round-glass`。
- 仅 `effect="glass" + shape="round" + theme="tag"` 激活玻璃模式。
- 无效组合按普通模式渲染，不添加 glass class、SVG 或警告。
- 不增加背景图片、filter id、折射强度等 props。

### CSS variables

提供以下最小定制面：

- `--td-tab-bar-glass-bg-color`
- `--td-tab-bar-glass-border-color`
- `--td-tab-bar-glass-highlight-color`
- `--td-tab-bar-glass-shadow`
- `--td-tab-bar-glass-blur`
- `--td-tab-bar-glass-saturate`
- `--td-tab-bar-glass-edge-width`
- `--td-tab-bar-glass-active-bg-color`
- `--td-tab-bar-glass-active-border-color`

### 能力边界

- 基础效果使用 LESS/CSS：透明基底、blur、saturate、边缘、高光、阴影和选中胶囊。
- Chromium 使用组件内部原创 SVG filter 做轻微边缘折射。
- SVG 增强不复制背景、不加载外部素材、不引用外部 filter id。
- 不引入 Canvas、WebGL、Three.js、动画库或新运行时依赖。
- 不使用持续 pointer tracking、ResizeObserver 或空闲状态 `requestAnimationFrame`。
- 不启用无限流光动画；只保留选择和按压状态过渡。
- reduced-motion 下关闭 transform 和 transition。

## 6. 两仓库实现要求

### `tdesign-common`

- 在 TabBar 样式中实现半透明基底、背景模糊、顶部高光、边缘描边、轻微色散和悬浮阴影。
- 增加 ring-masked optics 层样式，使折射只发生在胶囊边缘。
- 选中项使用半透明内胶囊和品牌色内容；未选中项继续使用语义文字色。
- 在 `style/mobile/theme/_components.less` 提供 light/dark token 覆盖。
- 不支持 backdrop filter 时提高基础填充可见度，保证对比度。
- 修复 TabBar 当前重复的 safe-area 选择器，不做其他无关重构。
- 中英文 API 文档只新增对应 Demo 入口，不重排无关内容。

### `tdesign-mobile-vue`

- 更新 `type.ts`、生成式 props 文件和中英文文档中的 `effect` 定义。
- 计算严格的 `isGlass`，仅有效组合追加 `t-tab-bar--glass`。
- 客户端挂载后为每个实例生成唯一 filter id。
- 只在 glass 模式渲染 `aria-hidden` SVG defs 和 `pointer-events: none` optics 层。
- SVG 使用独立设计的静态程序位移场和小幅 `feDisplacementMap`；不得参考现有滤镜节点顺序和数值。
- SSR 和首次 hydration 不访问 `window`、`navigator` 等浏览器全局。
- 动态切换 effect 或 shape/theme 时正确增加、移除增强 DOM。
- 新增独立 `round-glass.vue` Demo，使用原创 CSS 色块、线条和内容作为可观察背景。
- 更新 `src/_common` 到 common PR 的精确提交，并验证全新 clone 能初始化该 submodule。
- 不提交任何无关 snapshot、格式化或文档改写。

## 7. 测试与验收

必须新增或验证：

- 默认 normal 模式没有 glass class、SVG 和 optics 层。
- 合法组合正确启用 glass。
- 无效组合保持普通模式。
- 动态切换 effect 可以清理和恢复增强层。
- 多个 TabBar 实例的 filter id 不重复。
- 原有点击、`v-model`、fixed、placeholder、safe-area 行为不回归。
- 装饰节点不可聚焦、不参与辅助技术树。
- reduced-motion 下没有缩放和过渡。
- Chromium 显示边缘折射；Safari/Firefox 显示完整 CSS fallback。
- light/dark、纯色/多色背景、390×844 与 430×932 视口均无文字遮挡或层级错误。
- 空闲状态无 rAF、无持续重绘、无监听器增长。

建议验证命令：

- common：`pnpm lint`、`pnpm test:unit`
- mobile-vue：TabBar 单测、`pnpm test:demo`、snapshot、`pnpm lint`、`pnpm build`、`pnpm site`
- 在全新 clone 中执行 submodule 初始化并重复关键构建。

交付截图至少包括：浅色、暗色、Safari/Firefox fallback、动态选中四组；另提供短录屏展示切换和 reduced-motion。

## 8. 提交顺序

1. 从 `tdesign-common/develop` 创建独立 feature branch并完成公共实现。
2. 推送 common commit，创建 common PR。
3. 从 `tdesign-mobile-vue/develop` 创建配套 branch，指向该 common commit。
4. 完成 API、增强层、测试和 Demo，创建 mobile-vue PR。
5. 两个 PR 互相链接，并关联 issue #2571。
6. PR 描述写明原创实现、浏览器能力矩阵、fallback、测试结果和零新增依赖。
7. 不声称 Safari/Firefox 支持真实 SVG 折射；不需要 Figma，代码 UI、Demo、截图和录屏构成 UI 设计交付。

## 9. 完成定义

只有同时满足以下条件才算完成：

- 两个 PR 均可独立审查且互相正确关联。
- normal 模式零视觉和 DOM 回归。
- glass 模式在 Chromium 有原创边缘折射，在其他浏览器有完整 fallback。
- 暗色、reduced-motion、SSR、多实例和动态切换均经过验证。
- 没有第三方代码、SVG、素材或参数被直接复用。
- 没有背景复制、外部 filter id、Canvas/WebGL 或新增运行时依赖。
- 所有相关测试、lint、构建、Demo 和截图验证完成。
