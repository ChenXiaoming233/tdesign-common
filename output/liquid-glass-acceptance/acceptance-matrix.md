# TabBar Liquid Glass 验收矩阵

更新时间：2026-07-24T21:16:00+08:00

状态定义：`PASS` 已通过；`FAIL` 未达到门槛；`BLOCKED` 受外部条件阻塞；`PENDING` 尚未进入该阶段。

## 阶段一：Clean-room 基线与规格冻结

| ID    | 强制项       | 状态 | 验收证据                                                                                                                                                 |
| ----- | ------------ | ---- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| S1-01 | 基线来源     | PASS | common `87824d0d280408303e350f3b7ec7736ae72728c6`；mobile-vue `7013c55b46b7e02e533b3c36d2aa3cff0debfd2f`；api `bb9855f9464720a2a278779f7c0f2d381bbbe79d` |
| S1-02 | 工作区隔离   | PASS | 三个仓库均位于 `codex/tab-bar-liquid-glass`；与对应 upstream 基线的 diff 文件列表为空                                                                    |
| S1-03 | 竞争代码隔离 | PASS | `evidence-matrix.md` 只包含公开行为、职责、风险和链接，不含代码片段、SVG 图或竞争参数                                                                    |
| S1-04 | 规格完整性   | PASS | `adr.md` 已冻结 API、SSR、fallback、多实例、resize、性能、视觉、失败路径与排除项                                                                         |
| S1-05 | 范围检查     | PASS | 两个正式实现 worktree 初始状态干净；当前工作区的 `docs/1_*`、`docs/2_*`、旧原型与 output 均不在实现分支 diff 中                                          |

阶段一完成度：**5 / 5，100%，PASS**

## 基线清单

| 仓库               | 工作区                                                                               | 分支                         | upstream                                                        | 基础 SHA                                   |
| ------------------ | ------------------------------------------------------------------------------------ | ---------------------------- | --------------------------------------------------------------- | ------------------------------------------ |
| tdesign-common     | `/Users/chenyixuan/Coding/tdesign-common-tab-bar-liquid-glass`                       | `codex/tab-bar-liquid-glass` | `https://github.com/Tencent/tdesign-common.git` / `develop`     | `87824d0d280408303e350f3b7ec7736ae72728c6` |
| tdesign-mobile-vue | `/Users/chenyixuan/Coding/tdesign-mobile-vue-tab-bar-liquid-glass`                   | `codex/tab-bar-liquid-glass` | `https://github.com/Tencent/tdesign-mobile-vue.git` / `develop` | `7013c55b46b7e02e533b3c36d2aa3cff0debfd2f` |
| tdesign-api        | `/Users/chenyixuan/Coding/tdesign-common/output/liquid-glass-acceptance/tdesign-api` | `codex/tab-bar-liquid-glass` | `https://github.com/TDesignOteam/tdesign-api.git` / `main`      | `bb9855f9464720a2a278779f7c0f2d381bbbe79d` |

基线记录时间：2026-07-24T18:01:15+08:00

## 后续阶段门槛

| 阶段          | 强制验收范围                                                        | 状态    | 预计证据                                                   |
| ------------- | ------------------------------------------------------------------- | ------- | ---------------------------------------------------------- |
| S2 API 契约   | 默认值、合法值、非法值、生成幂等、变更范围、平台 lint/build         | BLOCKED | 本地 6/6 PASS；详见 `stage-2-results.md`；等待外部 PR 授权 |
| R0 来源门禁   | 授权状态、独立实现边界、历史结果归档                               | PASS    | `rework-baseline.md`                                       |
| R2 光学纹理   | Snell profile、厚度、bezel、IOR、specular、中心中性、DPR、预算     | PASS    | 29 项目标 Vitest；待补完整矩阵                              |
| R3 SVG filter | blur、位移、饱和、遮罩高光、两级 blend 的结构与 Chromium 实机验证  | PASS    | `rework-r3-*.png`；九个 primitive 顺序已验证                |
| R4 公共样式   | normal 回归、fallback、shape、bordered/safe/fixed、暗色、层级       | PENDING | 原结果仅为历史 SDF 实现，需重新验收                         |
| R5 Vue 集成   | mounted、SSR、hydration、多实例、resize、切换、卸载、失败路径、交互 | PENDING | 原结果仅为历史 SDF 实现，需重新验收                         |
| S6A 参数校准  | 完整控件、预设说明、候选对比、参数冻结、API 边界                    | PENDING | 参数清单、对比截图、校准记录、冻结常量                     |
| S6B Edge Dev  | 背景折射、中心稳定、四宽度、DPR、多实例、性能、静止状态、控制台     | PENDING | `output/playwright/` 截图、trace、浏览器报告               |
| S7 PR         | 范围、独立实现声明、API 同步、三仓 CI、关联 issue                   | PENDING | PR URL 与 CI 链接                                          |

## 阶段二：API 平台与公共契约

| ID    | 强制项        | 状态    | 验收证据                                     |
| ----- | ------------- | ------- | -------------------------------------------- |
| S2-01 | 默认值 normal | PASS    | 官方生成 props 与 Vitest 契约测试            |
| S2-02 | normal 合法   | PASS    | validator 测试                               |
| S2-03 | glass 合法    | PASS    | validator 测试                               |
| S2-04 | 非法值拒绝    | PASS    | validator 测试                               |
| S2-05 | 生成幂等      | PASS    | 两次 diff SHA-256 完全一致                   |
| S2-06 | 变更范围      | PASS    | 只影响 API 数据源和 Vue Mobile TabBar 生成面 |
| S2-07 | API 平台 PR   | BLOCKED | 尚未获得推送和创建外部 PR 的明确授权         |

阶段二本地完成度：**6 / 6，100%，PASS**  
阶段二总体状态：**BLOCKED，等待 API 平台 PR 发布授权**

## 阶段三：独立纹理生成器

| ID    | 强制项           | 状态 | 验收证据                                                                   |
| ----- | ---------------- | ---- | -------------------------------------------------------------------------- |
| S3-01 | 相同输入确定性   | PASS | Vitest 像素数组比较                                                        |
| S3-02 | 中心位移中性     | PASS | 中心 RGBA 断言                                                             |
| S3-03 | 水平方向         | PASS | 左右 R 通道方向相反                                                        |
| S3-04 | 垂直方向         | PASS | 上下 G 通道方向相反                                                        |
| S3-05 | 圆角透明         | PASS | 两张纹理角落 alpha 为 0                                                    |
| S3-06 | 边缘高光         | PASS | 边缘非零、中心为 0                                                         |
| S3-07 | 零尺寸           | PASS | 返回 null                                                                  |
| S3-08 | DPR 上限         | PASS | 最大为 2                                                                   |
| S3-09 | 像素预算         | PASS | 最大为 524288                                                              |
| S3-10 | 圆角上限         | PASS | 最大为短边一半                                                             |
| S3-11 | 调试预览         | PASS | Edge Dev 三幅画面非空；仅有站点 favicon 404 和既有插件 warning             |
| S3-12 | normal Demo 回归 | PASS | TabBar Demo 8/8，快照零变化                                                |
| S3-13 | 动态纹理预览     | PASS | 参数实时重建；390px 7.40ms、620px/DPR 2 20.90ms；背景运动 5 秒重建计数不变 |

阶段三完成度：**13 / 13，100%，PASS**

## 阶段六 A：正式 Demo 参数面板与光学校准

| ID     | 强制项             | 状态    | 验收证据                                                                          |
| ------ | ------------------ | ------- | --------------------------------------------------------------------------------- |
| S6A-01 | 最终稳定参数完整性 | PENDING | `effect`、`shape`、三项 CSS 变量及既有布局状态全部有控件                          |
| S6A-02 | 单值内部参数完整性 | PENDING | 厚度、bezel、IOR、位移、SVG blur、specular、光向、透明度、DPR 与尺寸均有控件      |
| S6A-03 | 控件元数据         | PASS    | 正式 Demo 显示当前值、默认值、范围和公开/内部属性                                |
| S6A-04 | 复杂预设说明       | PASS    | 曲面 profile、filter 图、编码、预算与 fallback 均列名并说明不可单值调节原因        |
| S6A-05 | API 边界           | PENDING | 调试参数不进入 Props、生产 DOM 或 API 文档                                        |
| S6A-06 | 对比矩阵           | PENDING | 网格、文字、图片、明暗主题、四宽度、两种 shape 均保存候选证据                     |
| S6A-07 | 参数冻结           | PENDING | 唯一默认参数同步到内部常量、ADR 与像素测试                                        |
| S6A-08 | 性能复验           | PENDING | 390px <=16ms、620px <=30ms、静止无持续重建                                        |

阶段六 A 完成度：**0 / 8，0%，PENDING**

## 阶段四：公共样式与降级效果

| ID    | 强制项              | 状态    | 验收证据                                                                     |
| ----- | ------------------- | ------- | ---------------------------------------------------------------------------- |
| S4-01 | normal 回归         | PASS    | 无 glass 层，原背景、圆角和边距保持不变                                      |
| S4-02 | 无 backdrop-filter  | PASS    | 强制 fallback 后文字清晰，背景透明度为 72%                                   |
| S4-03 | CSS fallback        | PASS    | Chromium 下 24px blur/160% saturation 生效；始终保留透明背景、边框和阴影     |
| S4-04 | normal shape        | PASS    | 计算样式为 0px 圆角、0px 左边距                                              |
| S4-05 | round shape         | PASS    | glass+round 使用 12px 水平、8px 底部、4px inset、32px 外圆角与 28px 选中胶囊 |
| S4-06 | bordered/safe/fixed | PASS    | hairline 规则未改；fixed 为 bottom 0；safe/fixed 组合选择器保留              |
| S4-07 | 暗色主题            | PASS    | 72% 深色基线、90% 白文字、24% 白边框，无纯白底                               |
| S4-08 | 层级与交互          | PASS    | 玻璃层 z-index 0、pointer-events none；TabBarItem z-index 1                  |
| S4-09 | 公共仓远程 CI       | BLOCKED | 尚未创建 PR，无法取得远程 lint/test/build 扇出结果                           |

阶段四本地完成度：**8 / 8，100%，PASS**  
阶段四总体状态：**BLOCKED，等待公共仓 PR 与远程 CI**

## 阶段五：Vue 组件生命周期与集成

| ID    | 强制项            | 状态 | 验收证据                                                       |
| ----- | ----------------- | ---- | -------------------------------------------------------------- |
| S5-01 | 默认模式          | PASS | 不创建玻璃类、层或 filter；8 个既有 Demo 快照零变化            |
| S5-02 | glass 挂载        | PASS | mounted 后创建三层玻璃 DOM 和 SVG filter                       |
| S5-03 | SSR               | PASS | 服务端只有 CSS baseline，不包含 filter 或客户端 ID             |
| S5-04 | hydration         | PASS | SSR baseline 水合无结构不一致警告                              |
| S5-05 | 多实例            | PASS | 每个实例生成唯一 filter ID                                     |
| S5-06 | resize 合并       | PASS | 同一帧三次 observer 回调仅调度一次重建                         |
| S5-07 | shape 切换        | PASS | DOM 更新后按新圆角重建纹理                                     |
| S5-08 | effect 切换       | PASS | glass 创建资源；normal 断开 observer、清空 filter 与增强层     |
| S5-09 | 卸载              | PASS | observer 断开且待执行 animation frame 被取消                   |
| S5-10 | Canvas 失败       | PASS | `getContext()` 失败时保留 CSS fallback，无未处理异常           |
| S5-11 | `toDataURL` 失败  | PASS | 编码失败时不保留失效 URL 或 filter，保留 CSS fallback          |
| S5-12 | 无 ResizeObserver | PASS | 不创建 SVG 增强且不注册全局 resize 监听                        |
| S5-13 | 交互回归          | PASS | 点击、受控值、二级菜单、`change` 与 `onChange` 共 8 项全部通过 |

阶段五本地完成度：**13 / 13，100%，PASS**  
阶段五总体状态：**PASS**；真实 Chromium 折射、性能和正式可交互 Demo 按计划在阶段六验收。

## 阶段一复验命令

```bash
# tdesign-common clean worktree
git status --short --branch
git diff --name-only upstream/develop...HEAD

# tdesign-mobile-vue clean worktree
git status --short --branch
git diff --name-only upstream/develop...HEAD

# tdesign-api clean clone
git status --short --branch
git diff --name-only upstream/main...HEAD
```
