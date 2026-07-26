# Liquid Glass Rework Baseline

更新时间：2026-07-24

## 来源门禁

- 行为参考：`archisvaze/liquid-glass@69f026af6da464337301941f8595484b89c88aae`。
- 参考仓库未发现 `LICENSE`、`COPYING` 或 GitHub License 元数据；不得向正式 TDesign 分支复制其源码、纹理算法、SVG 文本、参数表或资源。
- 正式实现使用独立的归一化 Snell 折射模型；参考仓库只用于本地行为对照。

## 返工状态

| 历史阶段 | 处理 | 原因 |
| --- | --- | --- |
| S1 Clean-room | 保留 | 分支、仓库职责和范围隔离仍有效。 |
| S2 API 契约 | 保留 | 首版公开 API 继续仅包含 `effect: normal | glass`。 |
| S3 纹理生成器 | 已替代 | 原 SDF edge-weight 算法不包含厚度、bezel、IOR 或曲面 profile。 |
| S4 公共样式 | 部分保留 | 层级和 fallback 保留；材质透明度和 CSS filter 职责重新冻结。 |
| S5 Vue 集成 | 部分保留 | 生命周期骨架保留；filter state 和 SVG graph 已替换。 |
| S6 校准 | 已作废 | 原 3px 位移及 CSS blur 结论不再作为交付证据。 |

## 当前默认实现

- 曲面：`squircle`
- 厚度比例：`0.7`
- bezel 比例：`0.42`
- 折射率：`1.5`
- 位移增益：`1`
- SVG blur：`0.4`
- 高光透明度：`0.5`
- 高光饱和度：`2`

这些值仅为内部默认值；正式 API 不暴露光学参数。后续 Edge Dev 校准矩阵是其唯一冻结依据。
