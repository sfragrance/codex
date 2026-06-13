# 合并上游分支操作指南

## Remote 配置

| 名称 | 仓库 | 用途 |
|------|------|------|
| origin | git@github.com:sfragrance/codex.git | 你的 fork（push 目标） |
| upstream | git@github.com:openai/codex.git | OpenAI 官方主仓库 |
| luca | git@github.com:LucaCappelletti94/codex.git | LucaCappelletti94 的 fork |

## 合并操作步骤

### 1. 拉取两个上游

```bash
git fetch upstream main
git fetch luca open-codex
```

### 2. 合并 openai/codex main

```bash
git merge upstream/main --no-edit
```

### 3. 合并 LucaCappelletti94/codex open-codex

```bash
git merge luca/open-codex --no-edit
```

### 4. 处理冲突

- `.github/workflows/` 下的文件：保持删除（我们只保留 `build-windows.yml`）
- 代码文件冲突：参考下方「核心功能保护」章节判断保留哪个版本
- 解决完冲突后：`git commit --no-edit`

### 5. Push

```bash
git push origin open-codex
```

## 核心功能保护：Flatten MCP Namespace Tools

相关 issue: https://github.com/openai/codex/issues/26234

### 背景

OpenAI 的 API 用 `{"type": "namespace", "name": "mcp__<server>", "tools": [...]}` 包装 MCP 工具，但第三方 provider（Ollama、LM Studio、OpenRouter 等）不认识这个结构。LucaCappelletti94/codex 的 open-codex 分支实现了 flatten 功能：对非 OpenAI provider 将 namespace 工具展开为标准 `function` 类型，命名为 `mcp__<server>__<tool>`。

### 关键文件（合并冲突时必须保留 luca 分支的改动）

| 文件 | 关键内容 |
|------|----------|
| `codex-rs/model-provider-info/src/lib.rs` | `namespace_tools: Option<bool>` 字段 |
| `codex-rs/model-provider/src/provider.rs` | `ProviderCapabilities.namespace_tools`、`resolve_namespace_tools()` 函数、相关测试 |
| `codex-rs/core/src/tools/spec_plan.rs` | `namespace_tools_enabled()`、`flatten_namespace_spec()` 函数、条件展开逻辑 |
| `codex-rs/core/src/tools/registry.rs` | `canonical_flat_tool_name` fallback 匹配逻辑 |
| `codex-rs/config/src/thread_config.rs` | config 中 `namespace_tools` 字段 |
| `codex-rs/config/src/thread_config/remote.rs` | remote config 中 `namespace_tools` 字段 |

### 冲突处理原则

1. 如果冲突涉及上述文件中 `namespace_tools` / `flatten` 相关代码 → **保留 luca 版本**
2. 如果上述文件有非 flatten 相关的冲突（如 upstream 新增了其他字段）→ 手动合并，确保两边改动都保留
3. 合并完成后验证：grep `namespace_tools` 和 `flatten_namespace_spec` 确认功能代码完整存在

### 合并后验证命令

```bash
grep -r "namespace_tools" codex-rs/model-provider-info/src/lib.rs
grep -r "resolve_namespace_tools" codex-rs/model-provider/src/provider.rs
grep -r "flatten_namespace_spec" codex-rs/core/src/tools/spec_plan.rs
grep -r "canonical_flat_tool_name" codex-rs/core/src/tools/registry.rs
```

如果官方 PR 合并了此功能，luca 分支的改动将不再需要，届时可以停止合并 luca remote。

## 注意事项

- 合并顺序：**先合并 upstream（官方），再合并 luca**。这样 luca 的 flatten 改动可以覆盖 upstream 如果有冲突
- 如果合并时出现 workflow 文件冲突，批量删除即可：
  ```bash
  git rm .github/workflows/<conflicting-files>
  ```
- 我们自己新增的 `build-windows.yml` 不会有冲突（上游没有这个文件）
