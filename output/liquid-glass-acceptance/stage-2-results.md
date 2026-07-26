# 阶段二验收结果：API 平台与公共契约

执行时间：2026-07-24  
本地验收：PASS  
外部 PR：BLOCKED，尚未获得推送与创建 PR 的明确授权

## 实施结果

- 在 `tdesign-api` 的 SQLite 权威数据源中新增一条 Vue Mobile TabBar Prop。
- 字段名为 `effect`，类型为 String，默认值为 `normal`，枚举为 `normal/glass`。
- 使用官方 `api:download` 导出 `packages/scripts/api.json`。
- 使用官方 `api:docs TabBar 'Vue(Mobile)'` 生成 API 仓产品文件。
- 使用官方 `finalProject` 模式将生成文件直接同步到 Vue clean-room worktree。
- 在 Vue TabBar 测试中增加默认值、合法枚举和非法枚举三个契约用例。

## 六项强制验收

| ID | 用例 | 状态 | 证据 |
|---|---|---|---|
| S2-01 | 不传 `effect` 默认 normal | PASS | 生成 props 默认值及 Vitest 断言 |
| S2-02 | `effect='normal'` 合法 | PASS | validator 单测 |
| S2-03 | `effect='glass'` 合法 | PASS | validator 单测 |
| S2-04 | 非法值被拒绝 | PASS | `other` validator 单测返回 false |
| S2-05 | 连续生成幂等 | PASS | 两次完整 binary diff SHA-256 均为 `4c43aa69512bfc0baaee9f70e43e6d8b677674f15e97b3c941edf4526f26d838` |
| S2-06 | 生成变更范围 | PASS | API 仓只变化数据库、API JSON 和 Vue Mobile TabBar 四个生成文件 |

本地完成度：**6 / 6，100%，PASS**

## 命令门禁

| 命令 | 状态 |
|---|---|
| `pnpm lint` in tdesign-api | PASS |
| `pnpm build` in tdesign-api | PASS |
| TabBar focused Vitest | PASS，1 file / 8 tests |
| `git diff --check` in tdesign-api | PASS |
| `git diff --check` in tdesign-mobile-vue | PASS |

## 当前变更范围

tdesign-api：

- `db/TDesign.db`
- `packages/scripts/api.json`
- `packages/products/tdesign-mobile-vue/src/tab-bar/props.ts`
- `packages/products/tdesign-mobile-vue/src/tab-bar/type.ts`
- `packages/products/tdesign-mobile-vue/src/tab-bar/tab-bar.md`
- `packages/products/tdesign-mobile-vue/src/tab-bar/tab-bar.en-US.md`

tdesign-mobile-vue：

- 四个对应 TabBar 生成文件
- `src/tab-bar/__test__/index.test.jsx`

## 已发现的上游漂移

API 平台的当前 TabBar 数据已经将 `onChange` 描述为 context 参数，而 `tdesign-mobile-vue/develop` 仍是直接 value 参数。官方 `finalProject` 生成因此同时带出以下既有差异：

- `onChange` 和 `change` 文档签名更新为 context。
- `TNode` 导入变为 type-only。

这些差异来自 `tdesign-api` 已合并的历史生成状态，并非本任务新增的数据记录。API 仓自身的本次 diff 不包含上述签名修改。正式提交 Vue PR 前必须在 PR 说明中标注该平台漂移，或与维护者确认是否由独立同步 PR 承担。

## 阶段门槛

本地实现和六项强制测试已经全部通过。由于尚未推送分支或创建 API 平台 PR，阶段二总体状态保持 `BLOCKED`；解除条件是获得外部发布授权并创建关联 PR。

