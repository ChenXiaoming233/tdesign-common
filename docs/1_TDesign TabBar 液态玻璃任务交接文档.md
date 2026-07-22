# TDesign TabBar 液态玻璃任务交接文档

本文用于记录背景、调研和决策来源；具体实现和验收以《TabBar 液态玻璃双仓库实现计划》为准。两者内容冲突时，以实施计划为准。

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
- 当前目录已经是 fork 的 `tdesign-common` Git checkout；执行 mobile-vue 配套实现时仍需准备独立的 `tdesign-mobile-vue` checkout。
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
- `effect="glass"` class 没有严格限制在悬浮胶囊所需的 round shape。
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
- 仅 `effect="glass" + shape="round"` 激活玻璃外壳；`theme` 只控制选中项使用普通内容态或 `tag` 内胶囊。
- 无效组合按普通模式渲染，不添加 glass class、SVG 或警告。
- 不增加背景图片、filter id、折射强度等 props。
- 不采用调用方提供背景副本、滤镜 ID 或背景定位的方案；这会要求业务同步滚动、尺寸、主题和动态背景，降低组件泛用性，并把浏览器实现细节暴露给调用方。

### CSS variables

新增以下 9 个 CSS variables 作为最小定制面；现有 TabBar CSS variables 保持不变：

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

- 根容器不得直接设置 `backdrop-filter`；使用 `::before` 基础材质层实现半透明基底、背景模糊和饱和度，避免建立 Backdrop Root 后阻断内部折射。
- 增加 ring-masked optics 层样式，使折射只发生在胶囊边缘。
- optics 与基础材质层同级绘制，明确基础材质、optics、TabBarItem 内容和 `::after` 的绘制顺序与 z-index：optics 位于基础材质之上、内容之下，内容位于所有装饰层之上；必须验证 optics 采样的是预期 backdrop，而不是已经完成的基础模糊层。不使用 `overflow: hidden` 裁剪现有二级菜单。
- optics 必须使用 `position: absolute`、`inset: 0`、`flex: none`，不得作为 flex item 参与 TabBarItem 宽度计算。SVG defs 宿主必须使用 `position: absolute`、`width: 0`、`height: 0`、`flex: none`、`pointer-events: none`，不得改变 TabBar 高度或占位高度；所有装饰伪元素和实体节点均不得捕获指针事件。
- `theme="tag"` 时选中项使用半透明内胶囊，并调整内胶囊不透明度、边缘亮度和品牌色内容；`theme="normal"` 保留现有普通选中内容态，只调整语义文字和图标颜色，不新增内胶囊。未选中项继续使用现有语义文字色。
- 在 `tab-bar/_var.less` 和 `tab-bar-item/_var.less` 声明公开 CSS variables，再在 `style/mobile/theme/_components.less` 提供 light/dark 覆盖。
- 默认先输出不依赖 `@supports`、CSS custom properties 或 `backdrop-filter` 的静态高不透明度基底，再输出使用 CSS variables 的覆盖声明；支持 custom properties 的浏览器使用后者，旧浏览器保留前者。仅在正向 `@supports` 检测到 blur 能力后降低基底遮蔽并启用 blur/saturate。optics 默认无填充、无滤镜，只有 mask、mask-composite 和 CSS blur 能力均满足时才启用 ring mask 与 SVG URL filter；标准实现使用 `mask-composite: exclude`，WebKit 实现使用 `-webkit-mask-composite: xor`，任一能力不满足时保持视觉中性。
- 修复 TabBar 当前重复的 safe-area 选择器，不做其他无关重构。
- 中英文 API 文档只新增对应 Demo 入口，不重排无关内容；`round-glass` 仅是 Demo 文件或文档块标识，不代表公共 API 或 `shape` 值，公共调用方式始终为 `shape="round" effect="glass"`。

### `tdesign-mobile-vue`

- 更新 `type.ts`、生成式 props 文件和中英文文档中的 `effect` 定义。
- 计算 `isGlass = effect === 'glass' && shape === 'round'`，theme 不参与增强 DOM 生命周期。
- 客户端挂载后使用 document 级共享计数器为每个实例生成唯一 filter id。优先使用 `Symbol.for('tdesign.tab-bar.glass-filter-id')`，当 `Symbol` 或 `Symbol.for` 不可用时使用长命名字符串键 fallback；两种键共享相同计数语义，避免多个 app、重复 bundle 或 HMR 实例发生 ID 冲突。
- 只在 glass 模式渲染 `aria-hidden="true"` SVG defs 和同时具备 `aria-hidden="true"`、`pointer-events: none` 的 optics 层；两者均不得参与 flex 布局。
- SVG 使用独立设计的静态程序位移场和小幅 `feDisplacementMap`；不得参考现有滤镜节点顺序和数值。
- 位移场几何使用 `objectBoundingBox` 或等效的归一化坐标；即使滤镜区域使用 `userSpaceOnUse`，也不得把位移参数绑定到固定 TabBar 宽高。同一套滤镜参数应覆盖 320px、390px、430px 和不同 DPR。
- SSR 和首次 hydration 不访问 `window`、`navigator` 等浏览器全局。
- 动态切换 effect 或 shape 时正确增加、移除增强 DOM；切换 theme 仅更新选中项样式。
- 新增独立 `round-glass.vue` Demo（该名称仅为 Demo 标识，不代表公共 API），使用原创 CSS 色块、线条和内容作为可观察背景。
- 更新 `src/_common` 到 common PR 的精确提交，并验证全新 clone 能初始化该 submodule。
- 不提交任何无关 snapshot、格式化或文档改写。
- 正式实现 SVG 增强前必须完成最小 Chromium 原型，覆盖 320px、390px、430px、浅色/暗色、多色背景、滚动场景和 DPR 1/2/3。只有在边缘折射可见、中心内容清晰、无接缝和明显尺寸差异时，才能进入正式实现；若不满足条件则暂停 SVG 实现并重新评估，不得以轻量边缘光学效果替代 Chromium 边缘折射目标。

## 7. 测试与验收

必须新增或验证：

- 默认 normal 模式没有 glass class、SVG 和 optics 层。
- `effect="glass" + shape="round"` 在 normal/tag theme 下均正确启用 glass。
- 无效组合保持普通模式。
- 动态切换 effect/shape 可以清理和恢复增强层，切换 theme 不重建外壳。
- 同一文档中的多个 TabBar 和多个 Vue app 实例的 filter id 不重复。
- Demo snapshot 中将动态 filter ID 归一化为稳定占位符；具体 ID 唯一性仅通过独立行为测试验证，不把递增序号写入长期 snapshot。
- 在不提供 `Symbol.for` 的模拟环境中，glass 模式仍能挂载并保持 CSS fallback 可读，不抛出运行时异常。
- `renderToString` 不输出增强 DOM，hydration 无警告，挂载后才增加 SVG 与 optics。
- 原有点击、`v-model`、fixed、placeholder、safe-area 行为不回归。
- glass 模式新增装饰节点后，TabBarItem 的 flex 计算、宽高、placeholder 高度和 safe-area 高度与基线一致。
- 装饰节点不可聚焦、不参与辅助技术树。
- reduced-motion 下没有缩放和过渡。
- 至少验证一个 Desktop Chrome stable 版本和一个 Android Chrome 或 Android WebView 环境，并在 PR 中记录具体版本；仅在这些已验证的 Chromium 环境中声明 SVG 边缘折射增强。Safari/Firefox 和其他未验证环境显示完整 CSS fallback。
- light/dark、纯色/多色背景、320×844、390×844 与 430×932 视口，以及 DPR 1/2/3 的代表性组合均无文字遮挡或层级错误。
- 空闲状态无 rAF、无持续重绘、无监听器增长。

建议验证命令：

- common：`pnpm lint`、`pnpm test:unit`
- mobile-vue：`npm run test:demo`、TabBar 单测、`npm run test`、`npm run lint`、`npm run build`、`npm run site:preview`
- common PR 触发的五个关联框架 lint/test/build/preview 必须全部通过。
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
- glass 模式在已验证的 Chromium 环境中有原创边缘折射，在 Safari/Firefox 和其他未验证环境有完整 fallback。
- 暗色、reduced-motion、SSR、多实例和动态切换均经过验证。
- 没有第三方代码、SVG、素材或参数被直接复用。
- 没有背景复制、外部 filter id、Canvas/WebGL 或新增运行时依赖。
- 所有相关测试、lint、构建、Demo 和截图验证完成。
