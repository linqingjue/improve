> **示例输出。** 这是 `/improve` 针对 [shadcn/ui](https://github.com/shadcn-ui/ui)
> 在提交 `1994caba0`（2026-06-10）上生成的一份真实计划，保留在这里用于展示格式。
> 该代码库此后已经发生变化——不要直接执行本计划；请在你自己的仓库中运行 `/improve`。

# 计划 001：提取 search 与 view 共用的 shadow-config 解析逻辑

> **执行代理说明**：逐步遵循本计划。运行每一条验证命令，并在进入下一步前确认
> 结果符合预期。如果触发“STOP 条件”章节中的任何情况，立即停止并报告，不得
> 自行发挥。完成后更新 `plans/README.md` 中本计划对应的状态行。
>
> **漂移检查（首先执行）**：`git diff --stat 1994caba0..HEAD -- packages/shadcn/src/commands/search.ts packages/shadcn/src/commands/view.ts packages/shadcn/src/registry/config.ts`
> 如果这些文件中任何一个自计划编写后发生变化，应先把“当前状态”摘录与实时代码
> 对照；只要不匹配，就视为 STOP 条件。

## 状态

- **优先级（Priority）**：P2
- **工作量（Effort）**：M
- **风险（Risk）**：MED
- **依赖（Depends on）**：无
- **类别（Category）**：tech-debt
- **计划基于（Planned at）**：提交 `1994caba0`，2026-06-10

## 为什么重要

`search.ts` 和 `view.ts` 都手写了相同的“shadow config”回退逻辑：先创建默认配置；
如果存在部分 `components.json`，则覆盖默认值；随后尝试完整的 `getConfig()`，失败时
回退到 shadow config。代码已经明确承认这种重复（`search.ts:31`：
“TODO: We're duplicating logic for shadowConfig here. Revisit and properly abstract this.”），
而且两份实现**已经发生漂移**：search 使用
`createConfig({style: "new-york", resolvedPaths: {cwd}})` 构造默认值，view 却从裸的
`configWithDefaults({})` 开始。以后任何针对部分配置处理的改动，例如新增默认值或
验证规则，都必须修改两次，并可能在无人注意时继续分化。

## 当前状态

- `packages/shadcn/src/commands/search.ts`——search/list 命令；约 91–115 行存在 shadow-config 代码块：

```ts
// search.ts ~91 (after `await loadEnvFiles(options.cwd)`)
// Start with a shadow config to support partial components.json.
// Use createConfig to get proper default paths
const defaultConfig = createConfig({
  style: "new-york",
  resolvedPaths: {
    cwd: options.cwd,
  },
})
let shadowConfig = configWithDefaults(defaultConfig)

// Check if there's a components.json file (partial or complete).
const componentsJsonPath = path.resolve(options.cwd, "components.json")
const hasComponentsJson = fsExtra.existsSync(componentsJsonPath)
if (hasComponentsJson) {
  const existingConfig = await fsExtra.readJson(componentsJsonPath)
  const partialConfig = rawConfigSchema.partial().parse(existingConfig)
  shadowConfig = configWithDefaults({
    ...defaultConfig,
    ...partialConfig,
  })
}

// Try to get the full config, but fall back to shadow config if it fails.
let config = shadowConfig
try {
  const fullConfig = await getConfig(options.cwd)
  if (fullConfig) {
    config = configWithDefaults(fullConfig)
  }
} catch {
  // Use shadow config if getConfig fails (partial components.json).
}
```

- `packages/shadcn/src/commands/view.ts`——view 命令；约 36–55 行采用相同模式，但从 `configWithDefaults({})` 开始，没有 style/cwd seed，这就是两份实现的漂移。
- `packages/shadcn/src/registry/config.ts:20`——包含 `configWithDefaults(config?: DeepPartial<Config>)`，是放置共享辅助函数的自然位置。同目录测试位于 `packages/shadcn/src/registry/config.test.ts`，新增测试应沿用其模式。
- 仓库约定：TypeScript ESM；使用 `@/src/...` 导入别名；zod schema 来自 `@/src/schema`；vitest 测试与源码同目录，命名为 `*.test.ts`。必须匹配 `registry/config.ts` 的风格。

## 所需命令

| 用途 | 命令 | 成功时预期结果 |
|------|------|----------------|
| 安装 | `pnpm install` | 退出码 0 |
| 测试 | `pnpm shadcn:test` | 全部通过 |
| lint + 类型检查 | `pnpm check` | 退出码 0 |

所有命令均从仓库根目录运行。

## 范围

**范围内**（唯一允许修改的文件）：
- `packages/shadcn/src/registry/config.ts`（新增共享辅助函数）
- `packages/shadcn/src/registry/config.test.ts`（为辅助函数新增测试）
- `packages/shadcn/src/commands/search.ts`（改为调用辅助函数）
- `packages/shadcn/src/commands/view.ts`（改为调用辅助函数）

**范围外**（即使看起来相关也不得修改）：
- `packages/shadcn/src/commands/init.ts`——它通过交互提示构建配置，不使用相同 shadow 模式，因此不存在该重复。
- `packages/shadcn/src/utils/get-config.ts`——`getConfig` / `createConfig` 保持不变；新辅助函数只组合调用它们。
- 不得改变完整 `components.json` 的解析行为；当完整配置存在时，两个命令必须与当前行为完全一致。

## Git 工作流

- 分支：`advisor/001-extract-shadow-config-resolution`
- 每个步骤分别提交；提交信息遵循仓库约定，例如 `refactor(cli): extract shadow-config resolution`。可参考 `git log` 中的 `feat(cli): improve search command`。
- 除非操作者明确要求，否则不得推送或创建 PR。

## 实施步骤

### 步骤 1：在 `registry/config.ts` 中新增 `resolveShadowConfig`

新增一个导出的异步函数：

```ts
export async function resolveShadowConfig(
  cwd: string,
  seed?: DeepPartial<Config>
): Promise<Config>
```

行为应从上方 `search.ts` 逻辑中提取：先构建
`configWithDefaults(createConfig({...seed, resolvedPaths: {cwd}}))`；如果 `cwd` 中
存在 `components.json`，使用 `rawConfigSchema.partial()` 对其进行部分解析并覆盖；
随后尝试 `getConfig(cwd)`，如果返回完整配置，则使用 `configWithDefaults(fullConfig)`；
如果抛出异常，保留 shadow config。`seed` 参数用于保留 search 的
`{style: "new-york"}` 初始化行为。

在 `config.test.ts` 中新增测试，并参照已有测试结构：没有 `components.json` 时返回
默认值；部分 `components.json` 正确覆盖；完整 `components.json` 走 `getConfig`
路径；完整配置格式错误时回退到 shadow config。

**验证**：`pnpm shadcn:test` → 全部通过，包括 4 个新增测试。

### 步骤 2：让 `search.ts` 改用辅助函数

用 `resolveShadowConfig(options.cwd, { style: "new-york" })` 替换约 91–115 行代码块。
删除不再使用的导入，例如 `createConfig`、`rawConfigSchema`，以及在文件其他位置不再
使用时的 `fsExtra` / `path`。

**验证**：`pnpm shadcn:test` → 通过；`pnpm check` → 退出码 0。

### 步骤 3：让 `view.ts` 改用辅助函数

用 `resolveShadowConfig(options.cwd)` 替换约 36–55 行代码块。不要提供 seed，以保留
当前裸默认值行为。清理未使用导入。

**验证**：`pnpm shadcn:test` → 通过；`pnpm check` → 退出码 0；
`grep -rn "shadow config" packages/shadcn/src/commands/` → 无匹配。

## 测试计划

- 在 `registry/config.test.ts` 中为 `resolveShadowConfig` 新增 4 个单元测试，使用步骤 1 列出的场景，并沿用该文件现有测试结构。
- 现有命令测试必须保持通过：`pnpm shadcn:test`。
- 本计划不新增集成测试；两个命令的行为按构造保持不变，只是把同一逻辑移动到一个位置。

## 完成标准

- [ ] `pnpm shadcn:test` 退出码为 0；`resolveShadowConfig` 的 4 个新增测试存在且通过
- [ ] `pnpm check` 退出码为 0
- [ ] `grep -rn "TODO: We're duplicating logic for shadowConfig" packages/shadcn/src/` 不返回匹配，说明重复代码及其 TODO 已删除
- [ ] `search.ts` 和 `view.ts` 都调用 `resolveShadowConfig`，二者均不再包含内联 shadow-config 代码块
- [ ] `git status` 显示范围外文件均未修改
- [ ] `plans/README.md` 中本计划状态已更新

## STOP 条件

出现以下情况必须停止并报告，不得自行发挥：

- 上述位置的实时代码与摘录不一致，说明自 `1994caba0` 后已经发生漂移。
- search 的 seed 差异（`style: "new-york"` 与 cwd resolved paths）和 view 的裸默认值差异，事实证明具有 `seed` 参数无法表达的关键行为；例如除非辅助函数加入命令专用分支，否则测试无法通过。
- 从任一命令移除代码块都要求修改范围外文件。

## 维护说明

- 以后需要支持部分配置的命令必须调用 `resolveShadowConfig`，不得复制该模式；审查者应拒绝新的内联 shadow-config 代码块。
- 如果 `init.ts` 以后加入部分配置恢复功能，也应考虑使用该辅助函数。本计划暂不包含，因为 init 的提示驱动流程具有不同语义。
- 审查重点：确认 view 在无 seed 路径上的行为逐字节一致。两份实现的漂移可能并非有意，但如果 view 的确依赖裸默认值，无 seed 调用能够保留该行为。
