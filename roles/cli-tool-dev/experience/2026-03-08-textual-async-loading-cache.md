# textual TUI 性能优化：异步加载 + 缓存

日期: 2026-03-08

## Triggers
- "textual" "@work" "thread" "LoadingIndicator" "卡" "阻塞" "性能" "缓存" "cache" "async" "exclusive" "call_from_thread"

## 背景

`gh-actions-tui` 初版所有 `gh` CLI 调用（`subprocess.run`）直接在 UI 线程执行。每次光标移动都触发 `gh run view` 请求（~1.5s），期间整个界面冻结，完全无法使用。

## 证据与现象

### 1. textual `@work(thread=True)` 是正确的异步模式

textual 8.x 中把阻塞操作移出 UI 线程的方式：

```python
from textual import work  # 注意：不是 from textual.work import work

@work(thread=True)
def _fetch_data(self) -> None:
    result = blocking_call()  # 在后台线程执行
    self.app.call_from_thread(self._render, result)  # 回到主线程更新 UI
```

**关键陷阱**：`from textual.work import work` 会报 `ModuleNotFoundError`。正确导入是 `from textual import work`。

### 2. `call_from_thread` 在 headless 测试中正常工作

`app.call_from_thread()` 在 `run_test()` 模式下也能正确调度回调。测试中需要等 worker 完成：

```python
async def wait_workers(app, pilot, timeout=15):
    while ...:
        await pilot.pause()
        active = [w for w in app.workers if not w.is_finished]
        if not active:
            return
        await asyncio.sleep(0.1)
```

`pilot.pause()` 只处理事件队列，不等待 worker 线程。必须轮询 `app.workers` 状态。

### 3. `exclusive=True` + `group` 实现请求防抖

快速移动光标时，每次 `row_highlighted` 都触发 jobs 请求。用 exclusive worker group 自动取消旧请求：

```python
@work(thread=True, exclusive=True, group="jobs")
def _fetch_jobs(self, run_id: int) -> None:
    ...
```

效果：连按 5 次 down（每次间隔 50ms），只有最后一个请求被执行。总耗时从 ~7.5s 降到 ~1.9s。

### 4. 缓存 + 过期渲染保护

简单的 `dict[run_id, data]` 缓存效果显著：

| 操作 | 无缓存 | 有缓存 |
|------|--------|--------|
| 切换到已访问 run 的 jobs | ~1.5s | **0.10s** |
| 重新打开同一 log | ~1.5s | **0.02s** |

但异步 + 缓存引入竞态：worker 完成时用户可能已经切到另一个 run。必须在渲染前检查：

```python
# 在 worker 回调中
if self._selected_run_id == run_id:
    self.app.call_from_thread(self._render_jobs, run_id, jobs)
```

### 5. LoadingIndicator 的显示/隐藏模式

textual 内置 `LoadingIndicator` widget。用 `display` 属性切换，和数据 widget 互斥：

```python
# compose 时两个都 yield
yield LoadingIndicator(id="runs-loading")
yield DataTable(id="runs-table", cursor_type="row")

# mount 时隐藏数据 widget
self.query_one("#runs-table").display = False

# 数据到达后切换
self.query_one("#runs-loading").display = False
self.query_one("#runs-table").display = True
```

## 关键决策

| 决策 | 依据 |
|------|------|
| 用 `@work(thread=True)` 而非 `asyncio` subprocess | textual 的 `@work` 自带 worker 管理（cancel/exclusive/group），不需要自己管 asyncio task |
| 全局 `DataCache` 而非 per-instance | 单文件工具，进程内只有一个 App 实例，全局 cache 最简单 |
| `r` 刷新时清空整个 cache | 用户主动刷新 = 期望最新数据，部分清理增加复杂度但无收益 |
| 不做 TTL 过期 | CLI 工具生命周期短，手动 refresh 足够；TTL 增加复杂度 |

## 可复用模式

1. **textual 异步三件套**：`@work(thread=True)` + `call_from_thread` + `LoadingIndicator`。阻塞 IO 必须用 `@work`，否则 UI 冻结。导入路径是 `from textual import work`。
2. **exclusive worker group 做防抖**：高频触发的数据请求（如 cursor move），用 `@work(exclusive=True, group="xxx")` 自动取消旧的、只保留最新的。
3. **异步渲染的 stale check**：多个 worker 并发时，渲染前必须检查"数据是否仍然是当前需要的"，否则会覆盖正确的数据。
4. **headless 测试等 worker**：`pilot.pause()` 不等 worker 线程完成，必须轮询 `app.workers` 判断是否全部 `is_finished`。
