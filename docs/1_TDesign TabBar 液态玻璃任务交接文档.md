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

### 约束来源与分级

- **项目事实和兼容基线**：issue 的交付要求、双仓库职责、目标分支、submodule 关系以及 `tdesign-common` browserslist 来自当前项目和 issue，应作为实施前提重新核验。
- **向后兼容硬约束**：默认 `effect="normal"` 不回归，旧浏览器保持可读，现有布局、事件、safe-area 和辅助技术语义不被破坏。这些要求来自组件库兼容责任，即使 issue 未逐条明文列出也应保持。
- **本方案工程决策**：无背景复制、无外部 filter id、无 WebGL/WebGPU、无新增运行时依赖、无持续 pointer tracking 和 `window`/`document` 全局监听，是为了控制本次竞争 PR 的复杂度与风险，不代表 TDesign 项目全局禁止 Canvas、尺寸感知或一次性调度。Canvas 2D、单实例尺寸观察和一次性 rAF 仅可作为静态 SVG 原型失败后的受控 Plan B，并须满足按需重建、像素预算和资源清理。
- **质量验收项**：SSR/hydration、多实例、动态切换、reduced-motion 和性能验证属于本方案主动提高交付完整度的门槛，并非 issue 明文要求。
- **实现预算而非官方禁令**：公开变量数量、optics 层数量、唯一 id 算法和是否使用独立 worktree 不应被表述成项目政策；准确实现由原型、仓库现有工具和评审证据决定。

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
- 没有 reduced-motion、可观察的背景边缘位移增强和针对性测试。
- `effect="glass"` class 没有严格限制在悬浮胶囊所需的 round shape。
- 文档包含较多无关重写，增加评审噪声。

### 共同不足

- 缺少完整的跨浏览器能力矩阵。
- 缺少多实例、动态切换和 SSR/hydration 验证。
- 没有同时做到“可观察的背景像素位移增强”和“零背景复制”。
- 没有充分证明空闲状态性能及 fallback 可读性。

## 4. 开源调研成果

| 项目 | 可参考方向 | 不应移植部分 |
|---|---|---|
| [simple-liquid-glass](https://github.com/lucaperullo/simple-liquid-glass) | SVG 渐进增强、fallback、reduced-motion、可见性和测试意识 | 原有滤镜代码、位移算法、参数、UA 判断及完整组件 |
| [nikdelvin/liquid-glass](https://github.com/nikdelvin/liquid-glass) | CSS/SVG 分层和尺寸无关的组件思路 | Astro、Tailwind、anime.js 和原 SVG 生成代码 |
| [liquid-glass-react](https://github.com/rdev/liquid-glass-react) | 视觉表现和交互方向 | React 实现、滤镜图、鼠标跟踪和布局逻辑 |
| [liquid-glass-effect](https://github.com/kevinbism/liquid-glass-effect) | 纯 CSS fallback 层次 | 将普通 glassmorphism 当作背景折射 |
| [liquid-glass-vue](https://github.com/WXperia/liquid-glass-vue) | Vue 组件接口拆分方向 | 源码、SSR 不安全实现；仓库缺少完整 LICENSE 文件 |
| [liquid-glass-studio](https://github.com/iyinchao/liquid-glass-studio) | 仅用于视觉观察 | WebGL2/WebGPU，体积和复杂度不适合 TDesign |
| [liquid-dom](https://github.com/AndrewPrifer/liquid-dom) | renderer 与适配层分离、尺寸无关几何、按失效重绘等架构原则 | WebGPU/Canvas/HTML-in-Canvas 渲染管线、scene graph、SDF、shader、参数和素材；调研快照中核心 packages 未声明明确许可证 |
| [archisvaze/liquid-glass](https://github.com/archisvaze/liquid-glass) | Chromium 中使用 `backdrop-filter`、按组件尺寸生成边缘位移图、SVG `feDisplacementMap` 渐进增强 | 未声明许可证；不得移植其源码、折射 profile、Canvas 算法、滤镜节点序列、data URL、参数或 Demo 交互 |

浏览器结论：SVG URL filter 用于背景边缘位移只作为 Chromium 渐进增强。Safari、iOS、Firefox 和未验证环境不承诺背景像素位移；支持 `backdrop-filter` 时提供 CSS 玻璃材质，不支持时提供高不透明度可读基底。

开源来源结论：只有明确许可证且与本次范围相符的内容才可进一步评估复用；未声明或许可证不完整的项目只用于观察功能目标、架构原则和视觉现象，不得取用源码、SVG、算法表达、参数或素材。

## 5. 已锁定的技术决策

### Public API

- 新增 `effect?: 'normal' | 'glass'`，默认 `normal`。
- 保留 `shape?: 'normal' | 'round'`，不增加 `round-glass`；`shape` 决定轮廓，`effect` 决定材质。
- 仅 `effect="glass" + shape="round"` 激活玻璃外壳；`theme` 只决定选中项使用普通内容态或 `tag` 内胶囊。
- 无效组合按普通模式渲染，不添加 glass class、SVG 或警告。
- glass 有效组合在样式层屏蔽 TabBarItem 的 split hairline，但不修改 `split` prop、对应 class 或运行时状态；移除 glass 后原有分割线行为立即恢复，其他模式完全不受影响。
- 不增加背景图片、filter id、折射强度等 props。
- 不采用调用方提供背景副本、滤镜 ID 或背景定位的方案，避免把滚动、尺寸、主题同步和浏览器实现细节转嫁给业务。

### 定制面

当前方案选择 9 个公开 CSS variables，覆盖玻璃基底、边框、高光、阴影、blur、saturate、边缘宽度和选中内胶囊；现有 TabBar CSS variables 保持不变。9 个是经过当前职责划分得出的定制面预算，不是官方硬上限；实现中如确需新增，必须证明其具有独立、稳定的主题定制价值。变量的准确名称、职责和默认值以[实施计划](./2_TabBar%20液态玻璃双仓库实现计划.md#public-api)为唯一依据，不在本文重复维护。

### 三级能力边界

1. **可读基底**：不依赖 CSS custom properties、`@supports` 或 `backdrop-filter` 的高不透明度背景，覆盖旧浏览器；light/dark 都使用直接 CSS 属性，并保证 blur 能力块内的主题规则能够重新覆盖该静态背景。
2. **CSS 玻璃 fallback**：检测到 blur 能力后启用半透明基底、blur、saturate、边缘、高光、阴影和选中胶囊；Safari、Firefox 等环境停留在这一层也必须完整可用。
3. **Chromium 边缘位移增强**：在 CSS 玻璃材质之上尝试原创 SVG URL filter 背景边缘位移，位移场优先使用静态 SVG，必要时由 Canvas 2D 按尺寸生成；CSS 能力检测只是结构安全门槛，不能证明浏览器实际采样并移动了 backdrop 像素，因此只对已人工验证的 Chromium 版本作增强声明。

无论增强是否生效，CSS fallback 都必须独立存在。方案不使用 UA 判断、背景复制、外部素材、外部 filter id、WebGL、WebGPU、新运行时依赖、持续 pointer tracking、`window`/`document` 全局监听或无限动画。静态 SVG 是首选；若原型证明其无法在目标尺寸稳定产生可观察位移，可启用原创 Canvas 2D Plan B，仅生成内部位移/高光图，不读取或复制页面背景。Plan B 只有在客户端挂载且第三级语法门槛通过后才初始化，每个 glass 根节点最多一个 `ResizeObserver`，只在实际尺寸或 DPR 变化时按需重建，并可用一次性 rAF/nextTick 合并调度；DPR 变化通过可清理的 resolution media query 或仓库既有等效机制感知，不增加 window resize listener。必须设置像素预算；能力不可用、尺寸为零、预算超限或生成失败时直接保留 CSS fallback。卸载或退出 glass 时清理 observer、DPR 监听、待执行任务和位图引用。reduced-motion 下关闭 transform 和 transition。

## 6. 决策理由与阻塞门槛

- 采用 `effect` 而不是新增 `round-glass` shape，是为了保持形状和材质正交，降低未来扩展成本。
- 不复制背景，是为了保持组件对滚动容器、动态背景、主题和不同业务布局的泛用性。
- 不暴露 SVG 参数，是为了防止内部浏览器实现固化为公共 API。
- 静态 SVG 位移场必须优先使用归一化坐标或等效的尺寸无关设计，不能绑定固定 TabBar 宽高。若静态 SVG 原型失败，Plan B 允许根据实例尺寸生成内部位移图，但必须采用受控尺寸/DPR 重建、像素预算和跨尺寸验证，不能把固定截图或背景副本作为输入。
- backdrop 采样受绘制顺序和 Backdrop Root 影响，不能仅凭 CSS 层级推断结果。正式实现前必须先完成 Chromium 图层拓扑原型，比较 optics 位于基础层上方、下方等候选结构。
- 只有原型证明边缘区域存在随背景内容变化的可观察像素位移、中心清晰、无接缝，并在代表性尺寸、DPR、主题和滚动场景下稳定，才能进入生产实现。先验证静态 SVG；若失败，再验证满足上述资源边界的 Canvas 2D Plan B。两种增强均失败时必须回到 CSS glass fallback，不能用普通高光、渐变或静态色散冒充背景位移。

具体 class、私有 CSS variable、图层职责、fallback 声明顺序、测试矩阵和完成定义全部由[实施计划](./2_TabBar%20液态玻璃双仓库实现计划.md)维护。

## 7. 移交与交付边界

- 正式 feature branch 必须从最新 `upstream/develop` 创建，不能从当前包含内部计划文档 commit 的本地 `develop` 直接分支。
- 当前 checkout 保留计划文档和 docs 记录点；正式实现优先使用从 `upstream/develop` 创建的独立 Git worktree。worktree 是推荐的隔离方式而非产品完成条件；若使用其他流程，也必须保证功能分支祖先干净且不携带内部计划文档。
- 本文和实施计划仅作为本地记录点，不得进入官方竞争 PR；正式 PR 只包含功能、公共 API 文档、Demo、测试和必要生成物。
- 先提交 `tdesign-common` PR，再让 `tdesign-mobile-vue/src/_common` 固定到 common PR 的精确 commit；两个 PR 互相链接并关联 issue #2571。
- 两个仓库都不得带入无关 snapshot、格式化、文档重写或顺手重构。
- PR 描述必须写明原创实现边界、三级浏览器能力矩阵、测试结果和零新增运行时依赖。
- 不声称 Safari/Firefox 支持 SVG 背景像素位移；代码 UI、可运行 Demo、截图和录屏构成 UI 设计交付，不额外提交 Figma。
