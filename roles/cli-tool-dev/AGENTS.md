# cli-tool-dev

## 职责

开发面向开发者的 CLI/TUI 工具：

- 偏好单文件 Python 分发（顶部注释声明依赖）
- 关注交互体验和实用性
- 优先复用已有 CLI 工具（如 `gh`、`git`）作为数据源，避免重复实现认证/API 对接

## 使用的工具

- TUI 框架：`textual`（Python）
- 数据源：`gh` CLI、`git` 等系统工具（subprocess 调用）
- 开发/调试：`textual console`、`textual run --dev`

## 相关知识

- `AGENTS.md` — 多 agent 协作规则（worktree、禁止 push master、PR 流程）
- `base/principles/knowledge-loading.md`
- `base/knowledge-sedimentation.md`

## 角色知识索引

> 本索引是 agent 的**首要加载入口**。先读索引了解有哪些知识，再按需 `Read` 具体文件。不要一次性加载所有文档。
>
> 每条摘要必须包含**足够的关键词**，让 agent 能判断是否与当前任务相关。

### Skills
<!-- - `skills/<skill-name>/SKILL.md` — 关键词丰富的摘要（2-3 句） -->

### Principles
<!-- - `principles/xxx.md` — 关键词丰富的摘要（2-3 句） -->

### Insights
<!-- - `insights/xxx.md` — 关键词丰富的摘要（2-3 句） -->

### Experience（按需查阅，不常驻加载）

- `experience/2026-03-08-textual-tui-first-practice.md` — textual 8.x TUI 开发首次实践：API 版本陷阱（ComposeResult/update_cell_at）、`gh run view --log` 输出格式解析（tab 分隔、按 job name 过滤）、headless 测试（run_test + pilot）、Collapsible 折叠/展开、单文件工具代码组织。
- `experience/2026-03-08-textual-async-loading-cache.md` — textual TUI 性能优化：`@work(thread=True)` 异步加载（导入陷阱 `from textual import work`）、`call_from_thread` 回主线程、`exclusive=True` + `group` 请求防抖、`LoadingIndicator` 显示/隐藏模式、dict 缓存 + stale check 防竞态、headless 测试中轮询 `app.workers` 等 worker 完成。

### Questions（已知的未知）

- `questions.md` — 0 条待验证

### Tools

- [st1page/gh-actions-tui](https://github.com/st1page/gh-actions-tui) — GitHub Actions TUI 查看器：基于 textual 的左右分栏界面，可浏览 Runs、Jobs 列表并查看 Job Log，数据通过 `gh` CLI 获取。
