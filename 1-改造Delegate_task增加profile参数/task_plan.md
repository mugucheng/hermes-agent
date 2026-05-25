# Task Plan: delegate_task 增加 profile 支持，替代 `hermes chat -p` CLI 调用

## Goal
为 delegate_task 增加 `profile` 参数，使子 agent 能加载指定 profile 的 AGENTS.md、config.yaml、.env，从而替代 paper-processor 中所有 `hermes chat -p quality-checker` CLI 子进程调用。

## Current Phase
Phase 6

## Phases

### Phase 1: 代码探查 & 方案设计 ✅
- [x] 理解 cron `_job_profile_context()` 的 profile 切换机制（scheduler.py:151-212）
- [x] 理解 `_build_child_agent()` 的构造逻辑（delegate_tool.py:870-1170）
- [x] 理解 `delegate_task()` 的参数解析和 batch/single 模式分发（delegate_tool.py:1918-2000）
- [x] 确认 DELEGATE_TASK_SCHEMA 的字段结构（delegate_tool.py:2660-2801）
- [x] 确认三个 profile 的 AGENTS.md 内容和调用关系
- [x] 确认 `set_hermes_home_override` → ContextVar，线程安全
- [x] 确认 `_build_child_agent` 在主线程构造，天然串行
- **Status:** complete

### Phase 2: delegate_tool.py — 加 profile 参数 ✅
- [x] 2.1 DELEGATE_TASK_SCHEMA 加 `profile` 字段（top-level + per-task）
- [x] 2.2 `delegate_task()` 函数签名加 `profile` 参数
- [x] 2.3 registry.register lambda 加 `profile` 传递
- [x] 2.4 `_build_child_agent()` 加 `profile` 参数
- [x] 2.5 `_build_child_agent()` 内部实现 profile 切换：
  - normalize_profile_name → Path(profiles/<name>)
  - set_hermes_home_override(profile_home) — ContextVar 线程安全
  - 加载 profile 目录的 .env（临时，构造后恢复 os.environ）
  - 读取 profile 目录的 AGENTS.md 注入到 system prompt 顶部
  - AIAgent 构造后恢复 HERMES_HOME 和 os.environ
  - `from pathlib import Path` 已添加到文件顶部
- [x] 2.6 batch 模式：per-task `t.get("profile") or profile` 优先级
- **Status:** complete

### Phase 3: 线程安全 & 环境隔离 ✅ 已确认安全
- [x] 3.1 确认 `_run_single_child` 的线程模型 → ThreadPoolExecutor，但 `_build_child_agent` 在主线程构造
- [x] 3.2 确认 `set_hermes_home_override` → ContextVar，天然线程安全
- [x] 3.3 确认 .env 加载走 `os.environ` → 线程不安全，但只在主线程构造阶段临时加载即可
- **结论：profile 切换放在 `_build_child_agent` 内（主线程），天然串行，无需额外锁**
- **Status:** complete

### Phase 4: paper-processor AGENTS.md — 改调用方式 ✅
- [x] 4.1 SPAWN规则总览表：`hermes chat -p quality-checker` → `delegate_task(profile="quality-checker", ...)`
- [x] 4.2 Step 3.5 执行代码：改为 delegate_task 格式
- [x] 4.3 Step 3.5 reject修复流程：重跑质检改用 delegate_task
- [x] 4.4 Step 3.5 fallback链：简化为 delegate_task 重试，支持 model 参数 fallback
- [x] 4.5 Step 5 全量质检：两个 checker 改为 batch 模式并行 delegate_task(tasks=[...])
- [x] 4.6 清理所有 `hermes chat -p` 引用（0 残留）
- **Status:** complete

### Phase 5: 集成测试 ✅
- [x] 5.1 冒烟测试：schema、函数签名、prompt 注入（6/6 pass）
- [x] 5.2 修复 `get_hermes_home` 函数名（原 `_get_hermes_home` 不存在）
- [x] 5.3 修复 ContextVar token 保存/恢复机制（`set_hermes_home_override` 返回 token）
- [x] 5.4 Profile 目录解析测试：quality-checker profile 存在，AGENTS.md (5231 chars) 可读
- [x] 5.5 完整 prompt 生成测试：profile AGENTS.md 正确注入到 system prompt 顶部
- [x] 5.6 ContextVar override/restore 循环测试通过
- **Status:** complete

### Phase 6: 清理 & 文档 ✅
- [x] 6.1 移除 paper-processor AGENTS.md 中的 `hermes chat -p` 相关说明（Phase 4 已完成，0 残留）
- [x] 6.2 更新 SPAWN规则总览表为 delegate_task 格式
- [x] 6.3 Commit & push hermes-agent（`64fd20a5b`）+ hermes-config（`df8d132`）
- [x] 6.4 更新 papermind-hermes skill 架构图和 Migration 说明
- **Status:** complete

## Key Questions
1. ~~`_run_single_child` 是在 ThreadPoolExecutor 里跑的吗？~~ ✅ 是，但 `_build_child_agent` 在主线程构造
2. ~~`set_hermes_home_override` 是 context-var 还是全局变量？~~ ✅ ContextVar，线程安全
3. ~~AIAgent 构造时，哪些步骤依赖 HERMES_HOME？~~ → config.yaml 加载、AGENTS.md 加载走 `_get_hermes_home()`
4. quality-checker 的 AGENTS.md 如何注入？→ 在 `_build_child_system_prompt` 中读取 profile AGENTS.md 并拼入 system prompt，或在 AIAgent 构造时设 `skip_context_files=False` 并设 `TERMINAL_CWD`
5. fallback 链（换模型重试）怎么实现？→ 方案A：paper-processor 层循环重试 delegate_task（和现有 CLI 方式一致）。方案B：利用 AIAgent 的 `_fallback_chain`。倾向方案A，更简单可控

## Decisions Made
| Decision | Rationale |
|----------|-----------|
| profile 参数加在 delegate_task 层而非 _build_child_agent 内部 | 保持 _build_child_agent 职责单一，profile 切换在外层完成 |
| 复用 cron 的 resolve_profile_env 逻辑 | 已验证可用的 profile 解析路径，避免重复造轮子 |
| 优先保证单任务模式安全，batch 模式后续扩展 | 减少首次改动范围，降低引入线程安全问题的风险 |

## Errors Encountered
| Error | Attempt | Resolution |
|-------|---------|------------|
|       | 1       |            |

## Notes
- HERMES_HOME override 机制：`hermes_constants.py` 的 `set_hermes_home_override` 使用 context-var（线程安全），但 .env 加载走 `os.environ`（线程不安全）
- AIAgent 构造时 `skip_context_files=True` 会跳过 AGENTS.md 加载，带 profile 时必须改为 False 并指定路径
- 当前 max_spawn_depth=2，paper-processor(depth=0) → delegate_task quality-checker(depth=1) 不超限
- pdf-parser → paper-processor 仍保持 `hermes chat -p` CLI 方式，不在本次改造范围
