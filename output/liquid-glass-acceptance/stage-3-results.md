# 阶段三验收结果：独立纹理生成器

执行时间：2026-07-24  
阶段状态：PASS

## 实施结果

- 新增纯函数 `createTabBarGlassTextures`，只接收尺寸、圆角和 DPR，输出 displacement 与 highlight RGBA 数组。
- 算法不调用 Canvas、DOM、SVG、随机数或外部资源。
- 使用独立圆角矩形 signed-distance、有限差分法线和固定左上光照生成纹理。
- DPR 上限为 2，单张纹理上限为 524288 像素；超限时保持比例降低采样率。
- 新增开发环境独立纹理诊断路由，不改变普通 TabBar Demo DOM。
- 诊断页支持实时调整宽度、高度、圆角和 DPR，并用单次 `requestAnimationFrame` 合并参数更新。
- 可独立开关 SVG 背景运动；背景动画不触发纹理重建。

## 十项强制像素测试

| ID    | 用例           | 状态 | 结果                                          |
| ----- | -------------- | ---- | --------------------------------------------- |
| S3-01 | 相同输入确定性 | PASS | 两次完整结果逐值相等                          |
| S3-02 | 中心位移中性   | PASS | 中心 RGBA 为 `128,128,128,255`                |
| S3-03 | 左右方向相反   | PASS | 左侧 R 小于 128，右侧 R 大于 128              |
| S3-04 | 上下方向相反   | PASS | 上侧 G 小于 128，下侧 G 大于 128              |
| S3-05 | 圆角外透明     | PASS | displacement 与 highlight 的角落 alpha 均为 0 |
| S3-06 | 高光仅位于边缘 | PASS | 存在非零边缘高光，中心 alpha 为 0             |
| S3-07 | 零尺寸安全     | PASS | 宽或高为 0 时返回 null，不抛异常              |
| S3-08 | DPR 限制       | PASS | 输入 DPR 4 时实际 DPR 为 2                    |
| S3-09 | 像素预算       | PASS | 4000x1000、DPR 2 输入仍不超过 524288 像素     |
| S3-10 | 极端圆角       | PASS | 圆角限制为短边的一半                          |

完成度：**10 / 10，100%，PASS**

## 自动化门禁

| 验收项                 | 状态 | 结果                                       |
| ---------------------- | ---- | ------------------------------------------ |
| 阶段三像素测试         | PASS | 10 tests                                   |
| 阶段二 TabBar 契约回归 | PASS | 8 tests                                    |
| 普通 TabBar Demo 快照  | PASS | 8 tests，快照零变化                        |
| ESLint                 | PASS | 新增 `src` 文件无 warning/error            |
| Vue TypeScript         | PASS | `vue-tsc --noEmit`                         |
| 来源检查               | PASS | 无随机数、外部 URL、参考仓库或竞争 PR 标识 |
| diff 检查              | PASS | `git diff --check`                         |

## Edge Dev 诊断页

访问地址：

`http://127.0.0.1:4176/mobile.html#/tab-bar-texture?glass-texture-debug=1`

验收环境：`/Applications/Microsoft Edge Dev.app/Contents/MacOS/Microsoft Edge Dev`

页面同时显示：

- R/G displacement 纹理。
- 左上光照的边缘 highlight 纹理。
- 使用相同纹理完成的隔离 SVG 合成预览。
- 宽度、高度、圆角、DPR 和背景运动实时控件。
- 实际纹理尺寸、像素数量、单次生成耗时和累计重建次数。

浏览器结果：

- 390x64、DPR 1 初始纹理生成耗时 7.40ms，低于 16ms 门槛。
- 620x64、DPR 2 生成 1240x128（158720 像素）纹理，耗时 20.90ms，低于 30ms 门槛。
- 开启背景运动并等待 5 秒后，重建计数保持为 3，未发生持续纹理计算。
- 1280x900 与 390x844 视口均无控件、文字或纹理画面重叠和横向溢出。
- 控制台无 Demo 运行时异常；1 个 `/favicon.ico` 404 和 3 个重复插件 warnings 来自现有站点基线。
- 不带 `glass-texture-debug=1` 时路由会回退到普通 `/tab-bar`。
- displacement、highlight 和 SVG 合成画面均非空；合成结果中心稳定、边缘对称并受圆角裁切。

动态预览验收：**S3-13，PASS**

证据：

- `output/playwright/stage-3-texture-preview.png`
- `output/playwright/stage-3-dynamic-texture-preview.png`
- `output/playwright/stage-3-dynamic-texture-preview-mobile.png`
- `output/playwright/stage-3-dynamic-texture-preview.trace`
- `output/playwright/stage-3-edge-dev.config.json`

## 阶段三新增实现文件

- `src/tab-bar/liquid-glass-map.ts`
- `src/tab-bar/__test__/liquid-glass-map.test.ts`
- `site/mobile/dev/liquid-glass-texture-preview.vue`
- `site/mobile/router.ts` 中的开发环境诊断路由

## 边界

本阶段没有把纹理接入 TabBar、没有新增 SVG 运行时、没有修改公共样式，也没有声称已经交付完整液态玻璃组件。这些分别属于阶段四和阶段五。
