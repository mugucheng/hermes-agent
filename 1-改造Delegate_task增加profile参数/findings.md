# Findings: delegate_task profile 支持

## 1. Cron 的 profile 切换机制（scheduler.py:151-212）

**核心函数**: `_job_profile_context(job_id, profile)` — context manager

流程：
1. `normalize_profile_name(raw_profile)` → 标准化名称
2. `resolve_profile_env(normalized_profile)` → 解析为绝对路径
3. `set_hermes_home_override(profile_home)` → 设置 context-local override
4. 修改全局 `_hermes_home = profile_home`
5. 执行 job
6. `reset_hermes_home_override(override_token)` → 恢复
7. `os.environ` 按 snapshot 恢复（delta-based）

**关键限制**: 有 profile 的 cron job 串行执行，因为 `os.environ` 是进程级的

## 2. delegate_tool.py 的子 agent 构造

**`_build_child_agent()` 签名**:
- 接收 `override_provider, override_base_url, override_api_key, override_api_mode`
- 接收 `role` (leaf/orchestrator)
- 不接收 profile 参数

**AIAgent 构造**:
```python
child = AIAgent(
    skip_context_files=True,   # ← 跳过 AGENTS.md！
    skip_memory=True,           # ← 跳过 memory！
    ephemeral_system_prompt=child_prompt,  # ← 用 _build_child_system_prompt 生成
    ...
)
```

**`_build_child_system_prompt()`**: 根据 goal/context 生成系统提示，不读取任何 AGENTS.md

## 3. set_hermes_home_override 的实现

需要确认：是 threading.local / contextvars 还是全局变量？
- 文件：`hermes_constants.py`
- 如果是 contextvars → 线程安全
- 如果是全局变量 → 线程不安全，batch 模式需要串行化

## 4. paper-processor 中 hermes chat -p 调用点

| Step | 调用 | 参数 | 返回处理 |
|------|------|------|----------|
| 3.5 | `hermes chat -p quality-checker --provider <P> -m "<M>" -q "质检章节..."` | 工作子目录 | 读 section_quality_report.json 判 pass/reject |
| 5-qc-light | `hermes chat -p quality-checker --provider <P> -m "<M>" -q "JSON质检..."` | 3个JSON文件名 | 读 extraction_quality_report_light.json |
| 5-qc-core | `hermes chat -p quality-checker --provider <P> -m "<M>" -q "JSON质检..."` | 4个JSON文件名 | 读 extraction_quality_report_core.json |

**共同特征**:
- 都用 `terminal()` 同步阻塞调用
- 都指定 `--provider` 和 `-m`（来自 model_pool.json 的 checker 配置）
- 都需要 fallback 链（换模型重试）
- quality-checker 自己写报告文件到工作目录，paper-processor 读文件判断结果

## 5. _build_child_agent 的线程模型

**关键发现**：所有子 agent 在主线程构造（delegate_tool.py:2045-2086循环），然后才提交给 ThreadPoolExecutor。
- `_build_child_agent()` → 主线程执行
- `_run_single_child()` → ThreadPoolExecutor 工作线程执行

**影响**：profile 切换如果在 `_build_child_agent` 内完成（构造前切换、构造后恢复），则天然串行安全。
但 `set_hermes_home_override` 是 ContextVar，Python 的 contextvars 在 ThreadPoolExecutor 里会自动 copy context，所以即使多线程也安全。

## 6. set_hermes_home_override 实现

```python
# hermes_constants.py:15-27
_HERMES_HOME_OVERRIDE: ContextVar[str | object] = ContextVar("_HERMES_HOME_OVERRIDE", default=_UNSET)

def set_hermes_home_override(path: str | Path | None) -> Token:
    value = _UNSET if path is None else str(path)
    return _HERMES_HOME_OVERRIDE.set(value)
```

**结论：ContextVar → 线程安全**。每个线程有自己的 context 副本。
但 `.env` 加载走 `os.environ` → **线程不安全**。需要限制 .env 只在主线程构造阶段临时加载。

## 7. 改为 delegate_task 后的变化

| 维度 | hermes chat -p | delegate_task(profile=...) |
|------|---------------|---------------------------|
| 进程 | 独立子进程 | 进程内线程 |
| AGENTS.md | 自动加载（-p 指定 profile） | 需手动注入或改 skip_context_files |
| 隔离 | 天然进程隔离 | 共享 os.environ，需注意 |
| model/provider | CLI 参数指定 | delegation config 或 profile config.yaml |
| 返回 | stdout 文本 | JSON {results: [{summary: "..."}]} |
| 超时 | terminal timeout=300 | max_iterations 限制 |
