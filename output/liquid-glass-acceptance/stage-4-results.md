# 阶段四验收结果：公共样式与降级效果

执行时间：2026-07-24  
本地状态：PASS  
总体状态：BLOCKED，等待公共仓 PR 与远程 CI

## 实施结果

- 新增 `t-tab-bar--glass` 状态以及 `__glass-base`、`__glass-refraction`、`__glass-sheen` 三层结构样式。
- 玻璃层绝对定位、继承现有轮廓、`pointer-events: none`，TabBarItem 位于其上且背景透明。
- 无 SVG 增强时保留半透明背景、边框和阴影；Chromium 支持时追加 `blur(24px) saturate(160%)`。
- 新增三项稳定 CSS 变量：背景色、边框色和阴影；明暗主题分别提供默认值。
- 中英文 API 说明和中文设计说明已同步，首版不公开折射强度或纹理参数。
- `glass + round` 重新排版为屏幕内缩的悬浮胶囊：12px 水平间距、8px 底部间距、4px inset、64px 外层高度和 56px TabBarItem。
- 图标与文字采用紧凑纵向排布，选中项使用 28px 同心胶囊和品牌色 tint；normal 与非 glass 模式不受影响。

## Apple 设计对齐依据

- Apple Human Interface Guidelines 将 Tab Bar 定义为顶层导航，不承载屏幕局部操作。
- Apple WWDC25 设计系统说明建议手机布局使用靠近屏幕边缘但保留额外间距的 capsule，让 Liquid Glass 导航层浮于内容上。
- 官方说明要求减少自定义背景和硬边框，通过布局、分组与 tint 表达层级，并将材质应用于整个控件而不是内部视图。
- 本实现只借鉴这些公开设计原则，尺寸、CSS、选择器、选中态和验收页均为独立实现，不复用 Apple 或竞争 PR 的代码与素材。

参考：

- `https://developer.apple.com/design/human-interface-guidelines/tab-bars`
- `https://developer.apple.com/videos/play/wwdc2025/356/`

## 八项本地验收

| 用例                | 状态 | Edge Dev / 源码结果                                                           |
| ------------------- | ---- | ----------------------------------------------------------------------------- |
| normal 回归         | PASS | 无玻璃层；背景仍为容器色；圆角与边距均为 0                                    |
| 无 backdrop-filter  | PASS | 强制为 none 后仍有 72% 背景、边框、阴影和清晰文字                             |
| CSS fallback        | PASS | 支持环境计算为 `blur(24px) saturate(1.6)`，增强失败不影响基线                 |
| normal shape        | PASS | 未产生额外圆角或外边距                                                        |
| round shape         | PASS | 32px 外圆角、28px 选中胶囊；12/8/4px 外间距与 inset；64/56px 高度             |
| bordered/safe/fixed | PASS | bordered 源规则未改；fixed 计算为 `position: fixed; bottom: 0`；safe 规则保留 |
| 暗色主题            | PASS | 背景 `rgba(36,36,36,.72)`、边框 `rgba(255,255,255,.24)`、文字 90% 白          |
| 层级                | PASS | 玻璃层 z-index 0、pointer-events none；内容 z-index 1、背景透明               |

本地完成度：**8 / 8，100%，PASS**

## 自动化门禁

| 门禁            | 状态    | 结果                                             |
| --------------- | ------- | ------------------------------------------------ |
| 全量 lint       | PASS    | Stylelint、Prettier、ESLint、TypeScript 全部通过 |
| 全量 test       | PASS    | 28 files、548 tests 全部通过                     |
| Less 编译       | PASS    | TabBar 与主题入口成功编译，glass 选择器展开正确  |
| diff 检查       | PASS    | `git diff --check` 无错误                        |
| Edge Dev 控制台 | PASS    | 最终加载 0 error、0 warning                      |
| 公共仓远程 CI   | BLOCKED | 尚未创建 PR，暂无远程 build 扇出结果             |

## 验收证据

- `output/liquid-glass-acceptance/stage-4-style-harness.html`
- `output/liquid-glass-acceptance/stage-4-style-harness.less`
- `output/liquid-glass-acceptance/stage-4-style-harness.css`
- `output/playwright/stage-4-common-style-light.png`
- `output/playwright/stage-4-common-style-dark.png`
- `output/playwright/stage-4-common-style.trace`
- `output/playwright/stage-4-apple-aligned-desktop.png`
- `output/playwright/stage-4-apple-aligned-mobile.png`
- `output/playwright/stage-4-apple-aligned-mobile-dark.png`
- `output/playwright/stage-4-apple-aligned.trace`

验收页只用于本地公共样式验证，不进入正式 PR。访问地址：

`http://127.0.0.1:4180/output/liquid-glass-acceptance/stage-4-style-harness.html`

## 边界

本阶段没有向 TabBar Vue DOM 接入玻璃层、没有创建 SVG filter、没有引入生命周期逻辑，也没有扩展首版 API。真实折射、多实例和失败清理属于阶段五；完整参数面板与光学校准属于阶段六 A。
