# TabBar Liquid Glass 竞争方案证据矩阵

记录时间：2026-07-24T18:01:15+08:00

## 使用边界

- 本文只记录公开 API 方向、用户可观察行为、文件职责和风险问题。
- 实现阶段不得从下列 PR 或参考仓库复制源码、DOM 结构、SVG 图、算法参数、选择器、测试文本或资源。
- 允许使用的依据只有 TDesign 当前 `develop`、TDesign 贡献规范、公开 Web API 规范和本项目 ADR 中的独立推导。
- `archisvaze/liquid-glass` 未检测到明确许可证，因此只作为沙盒视觉参照，禁止进入正式实现的来源链。

## 本地问题画像

- 目标：为移动端 TabBar 增加可选液态玻璃材质，并在 Chromium 中产生背景相关的真实折射。
- 交付边界：`tdesign-common` 负责公共样式；`tdesign-mobile-vue` 负责 Vue API 接线和客户端运行时；`tdesign-api` 负责生成型 API。
- 兼容约束：默认行为必须不变；非 Chromium、SSR 或运行时失败时保持可读、可操作的 CSS 基线。
- 竞争约束：可以借鉴问题拆分和验收维度，不能复用竞争实现代码。

## 公开证据

| 证据 | 可观察方向 | 需要规避的风险 | 允许用于本项目的内容 |
|---|---|---|---|
| [issue #2571](https://github.com/Tencent/tdesign-common/issues/2571) | 悬浮胶囊需要接近 iOS 液态玻璃的视觉，并要求 UI 与至少一个组件端实现 | 需求描述没有定义 API、兼容和性能标准 | 产品目标、竞赛期限、交付最低要求 |
| [common #2601](https://github.com/Tencent/tdesign-common/pull/2601) | 尝试提供较强的动态玻璃表现 | 变更面大，运行时与清理成本需要单独审查 | 需要关注动态效果成本这一问题 |
| [common #2609](https://github.com/Tencent/tdesign-common/pull/2609) | 同时调整文档、主题与基础玻璃样式 | CSS 玻璃外观不等同背景像素折射 | 主题、文档和 fallback 都属于交付范围 |
| [common #2628](https://github.com/Tencent/tdesign-common/pull/2628) | 将玻璃效果与圆形外观组合 | 将材质绑定为形状会限制 API 扩展 | 材质与形状需要明确关系 |
| [common #2635](https://github.com/Tencent/tdesign-common/pull/2635) | 增加玻璃样式及变量 | 仅公共样式无法证明真实折射或生命周期正确 | 公共样式和运行时必须分仓负责 |
| [common #2638](https://github.com/Tencent/tdesign-common/pull/2638) | 采用独立材质属性表达普通或玻璃效果 | 不能复用其命名之外的实现细节 | `effect` 作为公开材质维度的产品方向 |
| [common #2643](https://github.com/Tencent/tdesign-common/pull/2643) | 补充 TabBar 与 TabBarItem 的玻璃视觉 | 组件子项改动可能扩大 normal 模式回归面 | 内容层级必须纳入回归检查 |
| [mobile-vue #2254](https://github.com/Tencent/tdesign-mobile-vue/pull/2254) | 通过 Vue Demo 展示更丰富的折射路线 | Demo 级运行时不等于组件级生命周期保障 | 必须提供可交互、可测量的 Demo |
| [mobile-vue #2264](https://github.com/Tencent/tdesign-mobile-vue/pull/2264) | 将玻璃效果接入组件 API 和渲染 | API、DOM 和共享样式可能出现跨仓不同步 | 两个实现 PR 必须互相锁定提交 |
| [mobile-vue #2268](https://github.com/Tencent/tdesign-mobile-vue/pull/2268) | 将材质属性、Demo、文档和公共子模块联动 | 生成文件不能脱离 API 平台手工维护 | API 平台是正式交付依赖 |
| [archisvaze/liquid-glass](https://github.com/archisvaze/liquid-glass) | 沙盒验证证明 Canvas 纹理与 SVG 背景滤镜路线在 Chromium 可产生正确折射 | 无明确许可证；源码、图结构、参数和资源不可复用 | 仅作为“真实背景折射可行”的视觉证据 |

## 独立实现准入规则

正式实现只有同时满足以下条件才可开始：

1. 设计决策能够追溯到当前 TDesign 代码、贡献规范、公开平台 API 或本项目独立推导。
2. 代码作者不以竞争 PR diff 或参考仓库源码作为编码模板。
3. 新增标识符、DOM 分层、纹理公式、参数和测试断言均由 ADR 定义。
4. 最终评审仅对照行为矩阵验证目标，不从竞争源码移植修复。

## 调研路径

- 使用 GitHub CLI 查看 issue、PR 状态和文件职责面，没有提取或保存竞争源码。
- 检查本地 `tdesign-common` 与 `tdesign-mobile-vue` 当前组件入口、贡献规范和生成文件声明。
- 检查 `TDesignOteam/tdesign-api` 的公开 README 与生成命令。
- 未使用子代理：任务集中在三个明确仓库，且当前协作约束不允许委派。

