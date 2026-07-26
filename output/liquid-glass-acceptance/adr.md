# ADR：TDesign Mobile TabBar Liquid Glass v1

状态：Accepted for implementation  
日期：2026-07-24

## 目标

在 Chromium 中为 TabBar 提供背景相关的边缘折射，同时确保默认模式、SSR、非支持环境和失败路径仍保持现有 TabBar 的可读性与交互能力。

## 公共契约

- 新增 `effect?: 'normal' | 'glass'`，默认 `normal`。
- `effect` 表示材质，`shape` 表示轮廓，两者正交；四种组合均合法。
- v1 不公开折射强度、光照方向、纹理精度或动画参数。
- 稳定 CSS 变量限定为 `--td-tab-bar-glass-bg-color`、`--td-tab-bar-glass-border-color`、`--td-tab-bar-glass-shadow`。
- 动态 SVG filter 引用通过内部 CSS 变量传递，不属于公共定制 API。
- 调试 Demo 的参数控件不构成公共 API；校准完成后仍只保留 `effect`、既有 `shape` 与上述稳定 CSS 变量。

## 仓库职责

- `tdesign-api`：维护 `effect` 的 schema，并生成 Vue Mobile 的类型、props 与中英文 API 文档。
- `tdesign-common`：维护玻璃状态、三层视觉结构、fallback、主题变量和设计说明。
- `tdesign-mobile-vue`：维护独立纹理算法、Canvas 编码、SVG filter、生命周期、组件 DOM、Demo 和测试。

## 渲染结构

- `normal` 模式不创建任何玻璃节点，保持当前 DOM 和布局。
- `glass` 模式在 TabBar 根节点内、TabBarItem 之前加入三个绝对定位且不可交互的层：基线层、折射层、高光层。
- 内部类名固定为 `__glass-base`、`__glass-refraction`、`__glass-sheen`；内容通过现有 TabBarItem 层级位于其上。
- SSR 只输出 CSS 基线所需结构，不输出客户端 filter ID、Canvas 数据或 SVG 定义；客户端 mounted 后再增强。
- `glass + normal` 保留全宽矩形布局；`glass + round` 使用 12px 水平间距、8px 底部间距、4px 内容 inset 和 64px 外层高度。
- `glass + round` 的 TabBarItem 高度为 56px，使用图标上、文字下的紧凑排布；选中项使用 28px 同心胶囊和品牌色 tint，不添加独立材质层。

## 独立纹理算法

- 纯函数名为 `createTabBarGlassTextures`，输入 CSS 宽高、圆角与 DPR，输出位移和高光 RGBA 数组及实际采样信息。
- 基于圆角矩形 signed-distance 独立计算边缘距离与法线；中心保持中性，边缘带使用平滑插值产生位移。
- 位移图 R/G 分别编码水平与垂直方向，中性值为 128；B 固定为 128；有效区域 alpha 为 255，圆角轮廓外为 0。
- 边缘带宽为 `clamp(shortSide * 0.22, 8px, 16px)`；最大视觉位移为 `clamp(shortSide * 0.18, 8px, 12px)`。
- 高光由同一法线与固定左上光向独立计算，只保留边缘带，透明区域不参与混合。
- `MAX_DPR = 2`，`MAX_TEXTURE_PIXELS = 524288`；超限时保持宽高比降低采样率。
- Canvas 只将纯像素数组编码为 data URL，不参与距离、法线或高光计算。

## SVG 与 CSS 合成

- SVG filter 消费位移纹理，对背景输入执行二维位移，再以透明高光纹理进行 screen 混合。
- filter region 按最大位移向四周扩展，避免边缘裁切；颜色空间使用 sRGB。
- CSS 基线始终提供半透明背景、边框和阴影；支持 CSS backdrop blur 时追加模糊与饱和度。
- SVG 增强只有在纹理成功生成后启用；任一失败都移除增强状态而不移除 CSS 基线。

## 生命周期

- 私有 composable 名为 `useTabBarGlassFilter`，只在 `effect='glass'` 且 mounted 后运行。
- filter ID 在客户端生成，必须满足同页多实例唯一；SSR 输出不含 ID，因此不产生 hydration 差异。
- 使用单个实例级 `ResizeObserver`；同一帧的多次变化合并为一次 `requestAnimationFrame` 重建。
- 只有宽、高、圆角、DPR 或 effect 实际变化时重建；不使用连续动画、timer、pointer tracking 或全局监听。
- 切换 normal 或卸载时断开 observer、取消待执行帧并清空纹理字符串和增强状态。
- Canvas context、编码、ResizeObserver 或 SVG 支持失败时静默回退；开发模式允许每实例一次诊断警告。

## 性能和可访问性预算

- 390px 宽度单次生成目标不超过 16ms，620px 不超过 30ms。
- 单张纹理最多 512K 像素，DPR 最多 2；静止后不存在持续 CPU 工作。
- 所有玻璃层 `aria-hidden` 且 `pointer-events: none`，不得改变 tablist 语义、键盘行为、点击区域或对比度。

## 正式 Demo 参数面板与光学校准

- 阶段六正式组件 Demo 必须提供参数面板；现有阶段三纹理诊断页不继续扩展，避免对尚未接入真实背景的合成结果进行无效调参。
- 最终交付可提供的稳定参数必须全部能在 Demo 中调节，包括 `effect`、`shape`、三项稳定 CSS 变量及组件既有的 `fixed`、`safeAreaInsetBottom`、`bordered` 与 `placeholder` 状态。
- 可通过单一数值表达的内部参数应提供开发用滑块：厚度比例、bezel 比例、折射率、位移增益、SVG blur、高光透明度、高光饱和度、光照角度、基线透明度、纹理 DPR 与尺寸。
- 参数面板必须显示当前值、默认值、允许范围与是否属于公共 API；内部参数控件不得写入组件 Props、生产 DOM 或 API 文档。
- 宽度、主题、背景类型、多实例和运动背景使用分段控件、开关或选择器，不强行使用滑块。
- 无法通过调整单一值完成的项目保留为具名预设：曲面 profile、SVG filter 图结构、纹理通道编码、像素预算缩放策略、fallback 策略与高光混合模式。
- Demo 必须明确列出这些预设的名称、行为和不可直接调节原因，但不显示竞争实现参数或源码。
- 校准使用网格、文字、真实图片、明暗主题、四种宽度和 normal/round 形态；每组候选参数保存截图、性能数据和中心/边缘位移测量。
- 校准完成后只保留一套默认参数并写回内部常量、ADR 与像素测试；阶段六浏览器验收只验证已冻结参数。

## 排除项

- 不使用 WebGL、WebGPU、第三方运行时依赖、外部滤镜资源或调用方提供的 filter ID。
- 不实现连续动画、跟随指针折射、形态融合、动态背景取色或通用渲染器抽象。
- 不复用任何竞争 PR 或参考仓库的源码、SVG 图、参数、资源与测试文本；`archisvaze/liquid-glass` 未发现许可证，正式代码只独立实现公开可验证的光学行为。

## 交付顺序

1. API 平台契约。
2. `tdesign-common` 公共样式与文档。
3. `tdesign-mobile-vue` 纹理、生命周期、组件、Demo 与测试。
4. Edge Dev 实机验收后再提交最终 PR 证据。
