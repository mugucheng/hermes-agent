# Task Plan: delegate_task 加 model 参数 + pdf-parser→paper-processor 改造

## Goal
1. 给 `delegate_task` 增加 `model` 参数（暴露给 LLM），使 pdf-parser 能在调用时动态指定模型
2. 改造 pdf-parser AGENTS.md 的 Step 4，从 `terminal("hermes chat -p paper-processor ...")` 改为 `delegate_task(profile="paper-processor", model="...", ...)`
3. 这样整个 3-agent 管线（pdf-parser → paper-processor → quality-checker）全部走 delegate_task，零 CLI 依赖

## Current Phase
✅ Complete

## Phases

### Phase 1: 代码探查 & 方案设计 ✅
- [x] 1.1 理解 model 传递链：delegate_task → _load_config → _resolve_delegation_credentials → creds → _build_child_agent
- [x] 1.2 确认 _load_config 在 _build_child_agent 之前调用（读 parent config，非 profile config）
- [x] 1.3 设计方案：
  - model 格式：`"provider/model"` 或 `"<model>"`（无 provider 则用 profile/parent 的）
  - 解析用现有的 `resolve_runtime_provider(requested=provider, target_model=model)` 
  - 在 `delegate_task()` 中解析 model 参数后覆盖 creds 的对应字段
  - profile override 后如 model 为 None，从 profile config.yaml 读 `model.default` + `model.provider`
- [x] 1.4 关键函数：`resolve_runtime_provider(requested="bigmodel", target_model="glm-5-turbo")` 返回完整 credential bundle
- **Status:** complete

### Phase 2: delegate_tool.py — 加 model 参数 ✅
- [x] 2.1 DELEGATE_TASK_SCHEMA top-level properties 加 `model` 字段
- [x] 2.2 tasks[].properties 加 `model` 字段（per-task model override）
- [x] 2.3 `delegate_task()` 函数签名加 `model: Optional[str] = None`
- [x] 2.4 registry.register lambda 加 `model` 传递
- [x] 2.5 `delegate_task()` 主体解析 `model` 参数：
  - `"provider/model"` → split → `resolve_runtime_provider()` → 完整 credential
  - `"model-name"` → 仅覆盖 model 名
  - 优先级：model 参数 > delegation config
- [x] 2.6 `_build_child_agent()` 内 profile model fallback：
  - 当 model=None 且 profile 存在时，读 profile config.yaml 的 `model.default` + `model.provider`
  - 自动解析 provider credential
- [x] 2.7 batch 模式 per-task model override：
  - `t.get("model")` → 同样的 provider/model 解析逻辑
  - per-task model > top-level model > delegation config
- **Status:** complete

### Phase 3: pdf-parser AGENTS.md — 改 Step 4 ✅
- [x] 3.1 Step 4 从 `terminal("hermes chat -p paper-processor ...")` 改为 `delegate_task(profile="paper-processor", model="P_PROVIDER/P_MODEL", ...)`
- [x] 3.2 移除 `--provider` + `-m` 相关说明
- [x] 3.3 model_pool.json 读取逻辑不变，但输出格式改为组装 `model` 参数
- [x] 3.4 checker 模型传递：通过 `context` 参数传给 paper-processor
- [x] 3.5 Step 5 报告格式保持不变（已适配）
- [x] 3.6 清理所有 `hermes chat` 引用（0 残留）
- [x] 3.7 更新禁止事项列表（移除 CLI 相关禁止项）
- **Status:** complete

### Phase 4: max_spawn_depth 验证 & 调整 ✅
- [x] 4.1 调用链深度验证：Cron(depth=0) → pdf-parser → delegate_task paper-processor(depth=1) → delegate_task quality-checker(depth=2)
- [x] 4.2 pdf-parser config.yaml 加 `delegation.max_spawn_depth: 2`（允许 2 级嵌套）
- [x] 4.3 paper-processor config.yaml `delegation.max_spawn_depth: 1` ✅（只能 spawn quality-checker）
- [x] 4.4 quality-checker 无需 config（纯 leaf）
- [x] 4.5 `_get_max_spawn_depth()` 读当前 HERMES_HOME 的 config.yaml → 每个 profile 独立控制深度
- **Status:** complete

### Phase 5: 集成测试
- [ ] 5.1 语法检查：`py_compile.compile('tools/delegate_tool.py')`
- [ ] 5.2 Schema 验证：model 字段存在于 top-level 和 per-task properties
- [ ] 5.3 model 解析测试：`"bigmodel/glm-5-turbo"` → provider=bigmodel, model=glm-5-turbo
- [ ] 5.4 Profile fallback 测试：model=None 时从 profile config 读默认模型
- [ ] 5.5 pdf-parser AGENTS.md 残留检查：`hermes chat` → 0 matches
- **Status:** pending

### Phase 6: 清理 & 提交
- [ ] 6.1 Commit & push hermes-agent
- [ ] 6.2 Commit & push hermes-config
- [ ] 6.3 更新 papermind-hermes skill 架构图
- [ ] 6.4 更新 hermes-multi-agent skill（如需要）
- **Status:** pending

## Key Questions
1. `model` 格式用 `provider/model` 还是分别传 `provider` + `model`？
   → 倾向 `provider/model`，和 model_pool.json 的格式一致，减少 LLM 解析负担
   → 但需要解析逻辑（split `/`，查找 credential）
2. model 参数的 provider 需要完整 credential 解析吗？
   → 是的，需要从 auth.json 的 credential_pool 查 `custom:<provider>` 或 `<provider>`
   → 现有的 `_resolve_delegation_credentials` 可以复用
3. 当 profile 指定了 model 且 delegate_task 也指定了 model，谁优先？
   → delegate_task 的 `model` 参数优先（LLM 显式指定 > profile 默认）

## Decisions Made
| Decision | Rationale |
|----------|-----------|
| model 格式用 `provider/model` | 和 model_pool.json 格式一致，减少 LLM 拆分负担 |
| delegate_task model 参数 > profile config model | 显式指定 > 默认值 |
| provider 解析复用现有 credential 系统 | 避免重复实现 auth 解析 |

## Errors Encountered
| Error | Attempt | Resolution |
|-------|---------|------------|

## Notes
- 这次改造完成后，整个 3-agent 管线完全走 delegate_task，零 CLI 依赖
- pdf-parser 的 role 需要是 `orchestrator`（它需要 delegate_task 能力）
- 当前 max_spawn_depth=2，Cron(depth=0) → pdf-parser → paper-processor(depth=1) → quality-checker(depth=2) 刚好到顶
