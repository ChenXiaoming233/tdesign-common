# Liquid Glass Rework Results

更新时间：2026-07-24

## 已完成的返工阶段

| 阶段 | 结果 | 证据 |
| --- | --- | --- |
| R0 来源门禁 | PASS | `rework-baseline.md`；正式实现未复制无许可证参考仓库代码。 |
| R2 光学纹理 | PASS | `liquid-glass-map.test.ts` 覆盖 Snell profile、IOR=1、厚度单调性、bezel、specular、DPR 和预算。 |
| R3 SVG graph | PASS | Edge Dev 检查到 `feGaussianBlur → feImage → feDisplacementMap → feColorMatrix → feImage → feComposite → feComponentTransfer → feBlend → feBlend`。 |
| R4/R5 本地回归 | PASS | common lint、548 项 common 单测、30 项 mobile 目标单测及 mobile 构建通过。 |

## Edge Dev 结果

- 浏览器：Microsoft Edge Dev，Chromium 内核。
- 默认 390px：纹理 `366 × 64`，生成 `6.60ms`，总重建 `11.80ms`。
- 620px、纹理 DPR 2：纹理 `1192 × 128`，152,576 像素，生成 `9.80ms`，总重建 `16.20ms`。
- 独立 Edge Dev DPR 2 会话：`window.devicePixelRatio === 2`，创建一个实例级 SVG filter。
- 多实例：生成两个不同 ID，`t-tab-bar-glass-1` 与 `t-tab-bar-glass-2`。
- 移动背景：1.2 秒内背景位置约从 `8.87px` 变为 `33.40px`，重建计数保持 `2`。
- IOR=1 与 IOR=1.5 对照、surface=lip 对照已保存；前者无几何位移，后者恢复折射。

## 截图

- `output/playwright/rework-r3-grid-comparison.png`
- `output/playwright/rework-r3-ior-one.png`
- `output/playwright/rework-r3-ior-default.png`
- `output/playwright/rework-r3-surface-lip.png`
- `output/playwright/rework-r6-320-comparison.png`
- `output/playwright/rework-r6-390-comparison.png`
- `output/playwright/rework-r6-430-comparison.png`
- `output/playwright/rework-r6-620-comparison.png`
- `output/playwright/rework-r6-dpr2-comparison.png`
- `output/playwright/rework-r6-text-light.png`
- `output/playwright/rework-r6-image-light.png`
- `output/playwright/rework-r6-grid-dark.png`

## 未完成的外部项

- API、common、mobile-vue 远程 PR 和远程 CI 仍需要推送及创建 PR 的授权。
- 正式提交前需将 mobile-vue 的 `_common` 子模块指向 common 样式提交，并在三个 PR 中互相链接。
