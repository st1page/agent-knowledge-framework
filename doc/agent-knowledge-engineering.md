# Agent 知识工程：让 AI Agent 像人一样学习、记忆和成长（公开版）

> 通讯作者：[@tennyzhuang](https://github.com/tennyzhuang)

## 引入：Agent 不是菜，他只是不知道

很多时候 agent 产出的东西和预期有偏差、不可用，总需要反复提修改意见。

新人入职，即使经验丰富、技能多面，即使新工作的文档和代码组织得非常完善，实际上手时也需要一段时间 ramping up——上手摸索，和同事对齐。这个过程本质上是将同事的知识——包括那些不可见的知识——蒸馏给他。agent 和新人一样，能力不缺，缺的是关于你的项目、你的团队、你的偏好的那些上下文。给够了，就能上手。

很多人把 agent 用不好归结为"prompt 没写好"，然后去研究 prompt engineering——怎么措辞、怎么分步、怎么给 few-shot example。这些有用，但解决的是"怎么把一个问题问清楚"，而不是"agent 缺什么知识"。你可以用最完美的 prompt 让 agent 做一个跨层 API 重构，但如果他不知道项目四层架构各层的修改顺序、不知道 Python wrapper 有三处间接调用、不知道改完签名还要同步 .pyi 文件——还是会做错。prompt engineering 优化的是单次对话的表达效率，knowledge engineering 解决的是 agent 的认知基础。前者是战术，后者是战略。

agent 的可发挥上限很大程度上取决于你给他的知识工程。

## 知识从哪来？——让 agent 像人一样学习

不妨先问问人是怎么学会新东西的。面对一个新的库或工具，会直接进入文档把所有 API 背下来吗？似乎不是，背下来了也不敢说掌握了。而现在许多人对 skill 的使用就是这样——给 agent install 一个 skill，他只是知道了这些 API。但怎么用、有什么坑、什么情况用什么、注意什么，这些和场景息息相关的东西，不可能全由一个官方 skill 包罗万象。因此需要发挥 agent 能够和环境交互、获得反馈的优势，让他更主动地学习，就像人一样。重点是让 agent 用人的认知方式去主动学习和吸收知识。

### 复盘旧知：反思沉淀，将经历转化为存量经验

这是最基本的方式。每次人机交互中——人给出的信息、agent 做的决策、人给的反馈——都蕴含着人无法第一时间给到的知识。这些知识往往很重要，因为来源于 agent 在真实场景遇到的问题。完成任务之后先不退出 session，让 agent"把你觉得重要的东西沉淀到 AGENTS.md 里面"。

进一步，agent 可以反思。他能看到 session 之前的记录，也有自己的思考。最常用的 prompt 是"你刚才为什么没有 xxx？"——agent 会告诉你他以为的信息，由此可以把他不知道的那部分补充完整。有时候是 agent 犯蠢，那就"你反思一下，怎么改进 AGENTS.md 下次避免问题"。

### 对标他方：学习借鉴，从他人的实践中寻找参考

遇到棘手难题时，一篇命中问题的 blog 往往比官方文档更有价值。要给一个不了解的 codebase 加新功能，去看看有没有类似的 PR 往往是捷径。项目初期有多种选择时，去看看成熟的大型 codebase 怎么架构的，是非常有价值的参考。

这些作为程序员都很熟悉。但在 agent 协同时，习惯往往是自己搜到答案，再把结论喂给 agent——这样 agent 只拿到了"怎么做"，没拿到"为什么这么做"。不如直接让他去看原始材料，自己消化过的东西用起来比被告知的更扎实。确信从哪里可以获得答案时，不妨让 agent 自己去找。"去参考 https://github.com/apache/opendal/ 的结构用 PyO3 来组织项目的 python binding"

不只是代码和 PR，设计哲学类的文章同样有价值。比如 Erlang 社区的 "let it crash" 理念——与其到处写防御性错误处理，不如让进程崩溃、由 supervisor 重启。你可以直接让 agent 读原文，然后结合你的项目场景提炼出可落地的 principle：

```
读一下这篇文章 https://ferd.ca/the-zen-of-erlang.html
结合我们项目中 batch job 的错误处理现状，
思考 let it crash 的理念哪些可以借鉴、哪些不适用，
沉淀为一条 principle
```


关键不是让 agent "知道 let it crash 是什么"（他训练数据里肯定有），而是让他在**你的项目语境下**重新理解这个概念——哪些场景适用、边界在哪、和现有做法怎么衔接。这种"带着具体问题读文章"产出的 principle，比直接告诉他"我们要 let it crash"有用得多。

### 沙盒演练：模拟试错，通过"低成本实验"获取行为反馈

面对一个新工具，大多数人不是直接去读 API，简单了解之后通常直接下载下来，按预期的使用场景玩一玩，看看输出是否符合预期。agent 当然也可以这么做。

比如团队内部有一个集群调度工具 jobMan，没有公开文档，LLM 的训练数据里完全没见过。这时候与其让人写一份使用手册给 agent 读，不如直接让他去玩：

```
我们的环境里有一个内部集群调度工具 jobMan。
结合它的 help 和内部团队提供的文档，尝试使用它来提交一些不实际修改环境的 job，
包括但不限于：
1. 长时间运行的 job
2. 运行时间长短不一的 job
3. 使用不同资源用量的 job
4. 确定性会失败的 job，失败方式是 segment fault
5. 确定性会失败的 job，失败方式是 OOM
6. 确定性会失败的 job，失败方式是 Exit Code != 0
7. 部分 parameter 下会失败的 job
8. 50% 概率会失败的 job，足够重试可以成功
提交这些 case，并使用 jobMan 命令行去排查这些 job 的运行情况，
包括状态监控、日志查看、任务重试，以及做一些能让他们最终成功的尝试。
把你在这个过程中所做的所有尝试总结下来，沉淀为一份 skill。
```

这就是沙盒演练相比纯 LLM 的核心优势：agent 能实际运行命令、观察输出、根据反馈调整，通过与环境的交互来弥补训练数据的空白。对于这类内部工具，一次沙盒演练产出的 skill 可能比读十遍内部文档都有用。

沙盒演练的产出不只是"agent 学会了"，而是一份可复用的 skill 文档——prompt 最后一句就是"沉淀为一份 skill"。三种学习方式的产出最终都汇入同一个知识体系。

这里有一个贯穿全文的关键认知：**知识的具体内容应该让 agent 自己来写，而不是人替他写。** 不管是复盘、对标还是沙盒演练，产出的知识都是 agent 在真实场景中自己消化、自己总结的。agent 比人更知道自己需要记住什么——因为他是读者。人的角色是在犯错时问"为什么"、做对时让他"记下来"，而不是替他手写规则。

## 知识怎么存？——Persistent Memory

知识产生了，但 agent 的 context 窗口是有限的。一次对话结束，学到的东西大多会消失；即使在对话中，context 也会被压缩，早期信息会被摘要掉。如果知识只存在于对话里，agent 就是一个永远在 ramping up 的新人——每次开工都从零开始。

所以知识必须被持久化。但"存下来"只是第一步，更关键的是怎么组织——存的方式决定了知识能不能被找到、能不能被复用、会不会腐烂成垃圾。

### 存在哪？——四种载体

实践中，知识自然分布在四种载体上，各有分工：

**AGENTS.md / CLAUDE.md——随项目走的轻量指令**

最基本的实践。项目仓库根目录放一个 AGENTS.md（或 CLAUDE.md），agent 启动时自动加载，写的是"这个项目里 agent 必须知道的事"：协作规则、安全底线、关键路径。

特点是**始终在 context 里**，所以必须精简——只放"做错一次就会造成不可逆损失"的规则。比如"所有写操作必须在 worktree 中进行"、"agent 不得自行 merge PR"。

**独立知识库——agent 团队的共享大脑**

AGENTS.md 跟着项目走，但很多知识并不绑定某个项目——git worktree 的协作规范、GitHub PR 工作流的使用方式、跨层重构的修改顺序——这些是跨项目可复用的能力，需要一个独立于任何项目的地方来存放和共享。于是建了一个 `agent-knowledge-framework` 仓库，专门存放这类知识体系。

具体使用方式：在 `agent-knowledge-framework` 路径下启动 coding agent，开始时告诉他这次任务可能需要哪些角色的知识，然后让他去实际的工作目录（比如 `project-a`、`project-b`）干活。知识库是 agent 的"家"，工作目录是"工地"——从家里带上需要的知识，去工地干活，干完回来沉淀经验。

特点是**结构化、可检索、可演进**——不是扁平的笔记本，而是有明确分类和层次的知识库（下面会详细讲）。

**既有载体——文档和代码仓库本身**

每个项目里已经存在大量知识载体：内部文档说明了设计意图，项目公开文档定义了接口契约，架构文档记录了关键决策，PR description 解释了每次变更的上下文，代码本身体现了项目惯例。这些信息散落在不同的地方，但它们**已经在那里了**。

这一层的关键不是额外维护，而是**让 agent 知道这些载体的存在和位置**。独立知识库的作用之一，是充当索引——当 agent 需要理解某个决策的背景时，知识库能告诉它"去看某个 PR"或"去读某份架构文档"，而不是把所有内容都搬进知识库本身。

**人机协作——接入团队已有的知识网络**

上面说的是代码仓库和项目文档，但团队的知识远不止于此。Notion 上的设计文档、Slack/飞书里的技术讨论、Jira 里的需求背景、内部 wiki 上的 oncall 手册——这些才是团队日常沉淀知识的地方。如果 agent 只能看到代码仓库，它就像一个只读了课本、没参加过课堂讨论的学生。

读是基础——让 agent 能搜索 Notion 页面、查阅 IM 中被标记的技术决策、读取 issue tracker 里的上下文，这样它遇到问题时能自己去翻团队的讨论记录找到答案。

但更有价值的是**写**。agent 不应该只在终端里输出一堆文字然后等人来看。它可以把设计方案写进 Notion 页面让团队成员 review，在 IM 里汇报进展和遇到的问题，在 PR description 里用团队习惯的格式说清楚变更的上下文。当 agent 用团队熟悉的渠道和格式来展示工作时，它的产出才真正进入了团队的信息流，而不是锁在一个对话窗口里。

这一层的价值在于**让 agent 成为团队信息网络的参与者，而不只是旁观者**。能读，它就不是从零开始；能写，它才真正融入团队的协作循环。

### 独立知识库的内部结构：两个维度

四种载体中，AGENTS.md 足够简单不需要额外结构，代码和文档的组织由项目本身决定，团队知识网络由团队自身的工具链决定。真正需要设计的是独立知识库内部怎么组织。用两个正交的维度来切分：**横切按类型，竖切按受众**。

### 横切分类：五分法

知识的**类型**可以用五种覆盖：

| 类型 | 回答的问题 | 示例 |
| --- | --- | --- |
| **experience** | 发生了什么？ | "某个 workflow run 失败，排查发现是 `working-directory` 错误或 permissions 不足" |
| **skill** | 下次怎么操作？ | "跨层 API 重构的分层修改顺序：core → FFI → Python" |
| **principle** | 应该/不应该做什么？ | "每个功能用独立 worktree"、"集成测试必须跑" |
| **insight** | 为什么会这样？ | "迭代式 review 比一次性修复更可靠" |
| **question** | known unknown | "这个行为是否总是成立？需要什么证据验证？" |

它们之间有清晰的生成关系：

```
question（待验证的疑问）
    ↓（某次实践验证）
experience（原始素材）
    ↓（提炼）
skill / principle / insight（可复用知识）
```


experience 是基础。每次 agent 完成一项工作——修了 bug、跑通了流程、踩了坑——都先记录为 experience，然后从中提炼：

- 能抽出操作流程吗？→ skill
- 能抽出行为准则吗？→ principle
- 能归纳出跨场景的规律吗？→ insight

**只写 experience 不提炼，那只是日记。只写 skill 不留 experience，后人无法理解"为什么有这条规则"。两步都要做。**

为什么是五种不是更多？每多一种类型就多一个判断分支，分类的认知负担是沉淀流程能否持续运转的关键瓶颈。

**为什么没有 Fact（事实）？**

直觉上，"默认 runner 用的是某个镜像"、"某个 secret 只在特定环境可用"、"某些 workflow 只在特定 branch / path 条件下触发"这类系统状态描述似乎不属于现有五种中的任何一种——不是事件、不是流程、不是准则、也不是规律。

但 fact 几乎总是从 experience 中发现的——agent 在看某次 run 时确认了默认 runner，在排某个 job 失败时才知道 secret 或 trigger 条件的限制，这些发现过程本身就是 experience。

更关键的问题是：**fact 会过时，但没有自然的更新机制。** 系统升级、配置变更后，fact 库里的信息就变成了误导。而 experience 天然带时间戳，agent 查到旧记录时会自然意识到"这是半年前的，可能要验证一下"。单独维护 fact 列表，看起来方便，实际上是在制造一个没人维护就会腐烂的定时炸弹。

所以不单独维护 fact。环境信息放在项目文档或 AGENTS.md 里作为"环境描述"，维护责任跟着项目走；实践中发现的事实则自然沉淀为 experience，带着时间和上下文。

### 竖切分类：分角色

横切是类型维度，竖切是**受众维度**：这条知识是谁需要的？

```
agent-knowledge-framework/
├── base/                   # 跨角色通用知识
│   ├── principles/         # 通用原则（凭证安全、git worktree 协作）
│   ├── skills/             # 通用技能
│   └── insights/           # 通用洞察（跨层改动逐层 review）
└── roles/                  # 各角色目录
    ├── cli-tool-dev/       # CLI/TUI 工具开发
    │   ├── AGENTS.md       # 角色描述 + 知识索引（始终加载）
    │   ├── skills/         # 如"textual TUI 开发模式"
    │   ├── principles/     # 角色专有原则
    │   ├── insights/       # 角色专有洞察
    │   ├── experience/     # 如"textual TUI 首次实践踩坑记录"
    │   └── questions.md    # 待验证的疑问
    ├── maintainer/         # 知识仓库维护
    └── ...                 # 其他角色
```


判断标准不是"多个角色都有用就放 base"——db3-dev 和 db3-ops 都会接触同一个数据库，但关注的东西完全不同：一个关注怎么改 schema，一个关注怎么验证数据一致性。

比如 db3-ops 和 minio-ops 都会用 MinIO 客户端 `mc`，看起来是"同一个工具的知识"。但 db3-ops 用 `mc` 做数据治理——`mc find` 探测前缀是否存在、`mc mv` 迁移待清理数据，关注对象和元数据的一致性；minio-ops 用 `mc` 做安全审计——`mc admin trace` 抓取删除/覆盖请求、按 access key 和 IP 归因，关注谁在什么时候做了什么破坏性操作。同一个 `mc` 命令，两个角色 learn 出来的 skill 完全不同，硬合并到 base 反而丢失了各自视角的针对性。

**base 只放真正与角色视角无关的知识**——git worktree 协作规范、凭证安全原则、GitHub PR 工作流——不管从哪个角色的视角看都是一样的。

需要强调：**一个启动的 agent 不一定只对应一个 role。** 一次任务可能同时涉及数据库运维和 MDR 开发，可以同时加载多个角色的知识。role 不是 agent 的身份，而是知识的**组织单元**——给 agent 在读和写时提供更明确的指引：

- **读的指引**很好理解：告诉 agent 优先读哪个分类下的经验和技能，避免在整个知识库里漫无目的地搜索
- **写的指引**更需要 role 的预设：沉淀经验时，role 告诉 agent 从哪个视角去总结和提炼、更加注意什么。同一次跨层 bug 修复，mdr-developer 视角关注 FFI 层的类型映射陷阱，db3-ops 视角关注数据一致性的验证方法——提炼方向不同，产出的 skill 和 insight 也不同

为什么按角色切，而不是按项目、时间或主题？因为角色天然解决了"加载什么"和"怎么提炼"两个问题，不需要额外的过滤或分类逻辑。按项目切会把同一个角色的知识打散（一个角色可能跨多个仓库工作），按时间切则无法提供视角上的指引。

但角色划分不是一成不变的。一开始的分类多少有些随意——凭直觉划了几个角色，往里面填知识。真正重要的是**角色会随着知识库的自然增长而动态演化**：

- **分裂**：一个角色积累了太多知识，发现其中有两条明显不同的关注线——比如一个角色里既有 schema 设计的知识，又有大量数据治理的操作经验——这时候拆成两个角色比硬塞在一起更清晰
- **合并**：两个角色的知识高度重叠，维护两份 AGENTS.md 变成了负担，合并后反而更好找
- **互相学习**：一个角色踩过的坑，另一个角色可能也会遇到。dreaming 巡检时发现跨角色的相似 experience，可以提示将其提炼到 base，或者在另一个角色的 AGENTS.md 里加一条索引
- **自我修订**：角色的 AGENTS.md 里有一段 role description，描述这个角色关注什么、擅长什么。随着经验积累，agent 会发现最初的描述不再准确——实际工作范围比描述的更宽或更窄——这时候应该更新描述，而不是让描述和现实脱节

判断时机不需要刻意规划。当你发现一个角色的 AGENTS.md 索引变得臃肿、或者在两个角色之间反复犹豫"这条知识该放哪"、或者一个角色半年没有新增任何知识——这些都是重构的信号。角色划分的目标始终是**让 agent 更容易找到和沉淀知识**，当现有划分妨碍了这个目标，就该调整。

两个维度正交组合，就是完整的知识坐标系：**类型（五分法）× 受众（角色/base）**。

## 怎么维护？——让知识保持活性

知识存下来只是开始。不维护的知识库会迅速退化成垃圾场——过时的 experience 误导 agent，缺失索引的 skill 永远不会被读到，孤立的 insight 无法追溯到证据。核心矛盾是：**知识会腐烂，但人没精力时刻盯。**

### 反馈闭环——已有的知识没起作用怎么办

知识写下来了不等于 agent 就会用。经常遇到的情况是：规则明明写在那里，agent 还是犯了错。这时候不应该简单地重复"你去读一下 AGENTS.md"，而是要追问**为什么没起作用**，然后改进知识本身。

举个例子：worktree 规则早就写在 AGENTS.md 里了，但 agent 还是直接在 main 上改了代码。我问他"你为什么没用 worktree"，他反思后发现原因是"觉得只是先探索一下，不需要 worktree"——但探索很快变成了正式开发，再搬改动到 worktree 就很麻烦了。问题不是不知道规则，而是规则没覆盖到这个边界场景。于是他自己在 AGENTS.md 里补了一条："即使是'先探索一下'也应该在 worktree 里做"。

后来同一条规则又失效了——长 session 中 context 被压缩，agent 忘了 worktree 规则。这次原因不是规则写得不好，而是加载机制有缺陷。于是又加了一条："感觉不确定流程细节时，先重新读一遍 AGENTS.md"。

这就是反馈闭环：**犯错 → 追问为什么 → 发现是知识缺失/不够具体/加载时机不对 → agent 自己改进知识**。具体内容始终是 agent 自己写的，人的角色是在犯错时问"为什么"、做对时让他"记下来"。上层的组织结构——五分法、按角色分类——这些是人根据知识积累到一定量后的判断来设计的，但结构里面填什么内容，始终由 agent 产出。

### 元认知——沉淀知识本身也是一种知识

很早就发现，"怎么沉淀知识"这件事本身就需要被沉淀。

最初 agent 沉淀经验的方式是随手往 AGENTS.md 里加几行——没有分类、没有结构、没有提炼。几天之后 AGENTS.md 就变成了杂乱的备忘录，agent 自己都找不到之前记了什么。

于是有了 `knowledge-sedimentation.md`——一份沉淀规范，告诉 agent：完成重要工作后主动问自己四个问题（踩坑了吗→experience、能提炼出可复用的操作流程或模式吗→skill、有抽象准则吗→principle、能跨场景泛化吗→insight），判断知识放 role 还是 base，更新已有文档而不是新建文件。

这份规范本身也经历了迭代。最初只有"experience→skill/principle"的提炼路径，后来加入了 insight（从多次 experience 中归纳规律）和 question（known unknown）。每次发现沉淀流程有缺陷就更新规范——**这就是元认知：把关于知识的认知也纳入知识体系本身。**

更具体地说，元认知体现在三个层面：

- **怎么沉淀**（`knowledge-sedimentation.md`）：怎么分类、怎么提炼、怎么判断放哪
- **什么时候沉淀**：不是等人提醒才沉淀。规范里明确列出了触发沉淀的场景——PR review 修复（尤其多轮迭代的）、跨层 bug 修复、首次跑通某个流程、踩了非显而易见的坑。这些场景本身就是一种认知：知道"什么时候该停下来反思"比"怎么反思"更难习得
- **反思的意识**：agent 天然没有"该沉淀一下"的冲动，这个意识必须被显式写进规范。规范要求 agent 完成上述场景后主动问自己四个问题，而不是等用户说"总结一下"——把反思从被动响应变成主动习惯。在工具层面，Claude Code 提供了 hook 机制，可以在特定事件（如工具调用前后、session 结束时）确定性地触发脚本，理论上可以把沉淀检查挂到 session 结束的 hook 上。但 Codex 等其他 agent 框架目前没有类似机制，所以还是选择把触发条件写在 prompt/规范里——最通用的方式，不依赖特定工具

### Comment——欢迎来到 Web 2.0

experience 是某个时间点的快照，但知识不是静态的。一条 experience 可能因为系统升级而失效，可能被另一个角色从不同角度重新解读，也可能在后续提炼成了一条 insight。

如果 experience 之间没有连接，知识库就是一堆孤岛。于是引入了一个轻量的 comment 机制——在 experience 文件末尾加 `## Comments` 段落，记录后续发生的关联：

```
## Comments

### memory-cleaner (2026-03-04)
从本次和 db3-ops 的 SIGKILL 经验中提炼出通用 insight：
`base/insights/sigkill-skips-cleanup.md`

### db3-developer (2026-02-26)
本条经验在系统升级后部分失效，新行为见
`experience/2026-02-26-s3-last-modified-column.md`
```


comment 形成了 experience 之间、experience 与 insight/principle 之间的**双向链接**。读 experience 时能看到"这条经验后来发生了什么"；读 insight 时能追溯到"这条认知是从哪些 experience 归纳出来的"。

边界同样重要——comment 是批注和索引，不是正文。如果一条 comment 开始包含深度分析，它应该成为独立的 insight 或 skill，comment 里只留链接。

### Dreaming——仿生 agent 会梦见电子 5090 吗

人在睡眠中整理白天的记忆——agent 可以有类似的机制。于是设了一个 `maintainer` 角色，定期对知识库做自动化巡检：

**机械层（linter）——自动执行**：
- 坏链接扫描：Markdown 里引用的路径是否还存在
- 索引缺口：skill/insight 文件在目录里存在，但 AGENTS.md 索引里没有条目
- 近似重复检测：对 skill/insight 的标题和描述做 token Jaccard，相似度高则报候选

**语义层（synthesis）——输出候选，人来决策**：
- experience 里 `source:` 指向了 insight，但 experience 本身没有反向链接
- 多条 experience 描述相似现象，可能值得归纳为 insight
- experience 中提到的版本号/路径/命令明显过时

关键的设计决策：**linter 自动做，synthesis 只输出候选清单让人判断**。让 agent 自主做内容提炼和合并是危险的——可能误判两条看似相似但实际不同的 experience，或者把有价值的上下文抽象掉。dreaming 的价值在于**发现问题，而不是解决问题**。

除了巡检，dreaming 还有一个自然的用途：**复盘白天新增的 experience，并跑相关的 lab**。白天工作中沉淀的 experience 往往只记录了"发生了什么"和"怎么修的"，但边界条件没有被充分探索。夜间 agent 可以扫描当天新增的 experience，从中提取出值得验证的假设，设计最小实验去试——就像人白天经历了一件事，晚上做梦时大脑会反复模拟各种"如果当时不这样做会怎样"。产出的 lab 记录和修正后的 skill，第二天人来 review。

这也验证了元认知的论点：巡检规则本身就是一种被固化的认知——把"什么是知识库的不健康状态"从人的直觉变成了可重复执行的检查流程。

## Agent 怎么读？——Context Engineering

知识存好了、维护着，最后一个问题是：agent 怎么在对的时机读到对的知识。这是 context engineering——不是把所有知识都塞进 context，而是让 agent **按需加载**最相关的那些。

为什么不能全塞进去？一方面是 context 窗口有限，塞满了就没有留给思考和工作的空间。但更重要的原因是：**不相关的知识会污染 agent 的判断。** context 里放了一条"网络超时时应该重试 3 次"的 principle，agent 在处理一个完全无关的本地文件解析错误时也可能受其影响，开始加不必要的重试逻辑。知识越多不等于越好——精确地给到当前任务需要的知识，比把所有知识一股脑塞进去效果好得多。按需加载不只是节省空间，**精炼 context 本身就有价值**。

### 分层加载：常驻 vs 按需

不是所有知识都同等重要。加载策略分三层：

**常驻加载（每次任务都在 context 里）**：
- `AGENTS.md`：协作规则、安全底线、知识加载入口
- `roles/<role>/AGENTS.md`：角色职责 + 知识索引

这两份文件就像"工作记忆"——始终在线，决定了遇到问题时该往哪找。

**按需加载（通过索引命中后读取）**：
- skill / principle / insight：通过 AGENTS.md 索引摘要判断相关性后加载
- experience：作为"证据/边界/反例"在需要时升级加载

**触发式加载（特定场景自动提醒）**：
- 接触凭证 / 权限 / token → 先读 `base/principles/credential-safety.md`
- 任何写操作（多 agent 并行）→ 先读 worktree 相关规范

### Trigger：让知识主动找到 agent

被动的索引不够——agent 不知道自己不知道什么。但不能把所有知识的触发条件都塞进常驻 context，那样 AGENTS.md 会膨胀到不可维护。

实际的做法是两层过滤：

**第一层：AGENTS.md 摘要（常驻，轻量）。** 摘要本身就兼任 trigger——关键是写够关键词。比如 experience 索引里写"ClickHouse 写入 MinIO 的 PutObject 重试/499 调研"，当 agent 遇到 499 状态码时，扫一眼摘要就能命中这条记录。摘要不需要额外的 trigger 字段，它本身就是最轻量的触发器。

**第二层：文档 front matter（按需，精确）。** agent 打开文档后，front matter 里的 `triggers` 字段提供更精确的匹配：

```yaml
---
triggers:
  - "GitHub Actions"
  - "workflow"
  - "触发方式"
  - "needs DAG"
  - "timeout conclusion"
---
```


这一层不占常驻 context，只有 agent 已经打开文档时才会看到——用来确认"这个文档确实和当前任务相关"，或者在文档较多时辅助筛选。

两层配合的效果是：**摘要做粗筛（轻量、常驻），front matter 做精筛（精确、按需）**。不需要把所有 trigger 提前加载，摘要充当了第一层过滤。

对 experience 来说，这类**症状型 trigger** 尤其有价值——error message、异常现象这些具体线索，比抽象的主题描述更容易被精确命中，让 agent 直接跳到历史踩坑记录而不是从头排查。

AGENTS.md 里的"触发型 skill"是更强的版本——直接写成硬性规则："接触凭证/权限/token 前**必须**先读 `base/principles/credential-safety.md`"。不是建议，是 preflight checklist。

### 联想：从一条知识到一片知识

单点命中往往不够。读到一条 skill 后，可能还需要理解背后的 principle、或者回溯到产生这条 skill 的 experience。这就需要知识之间的**连接**：

- **纵向连接（提炼链）**：experience → skill/principle/insight，通过 `source` 字段和 "Escalate to experience if" 段落实现
- **横向连接（comment 网络）**：experience 之间的关联，通过 comment 段落中的链接实现
- **索引连接**：每条知识在 AGENTS.md 中的摘要自带丰富关键词，相似关键词的条目自然形成关联

这些连接让知识库从线性列表变成了一个**网络**——从任意入口进来，都能沿着连接找到相关上下文。

### Context 压缩后的恢复

长 session 中 context 会被自动压缩（compaction），压缩后 AGENTS.md 和角色 AGENTS.md 的内容可能被摘要掉。这时需要识别"忘了什么"：

出现这些信号就按 compaction 处理：
- 不确定当前角色的知识索引里有什么
- 不记得团队对 PR/GitHub Actions/文档发布 的硬性要求

恢复动作很简单：重新读一遍 `AGENTS.md` + `roles/<role>/AGENTS.md`，30 秒就能回到正轨。这也是为什么这两份文件必须精简——它们是 compaction 后的恢复锚点，太长了恢复成本就会变高。

## 结语

回到开头：agent 不是菜，他只是不知道。

本文介绍的五分法、分角色、comment 机制等结构并不是唯一的解法，只是在实践中迭代出来的一个例子。完全可以用不同的分类、不同的组织方式。**最重要的是第一步：让 agent 在干活的时候有地方把知识沉淀下来。** 哪怕一开始只是往 AGENTS.md 里随手加几行，也比什么都不记强一百倍。至于后面怎么组织、怎么分类、怎么加载——这些都是知识积累到一定量之后自然会面临的问题，到那时候重构成本很低，因为面对的只是一堆文本文件。

整套体系不依赖数据库、向量检索或复杂 pipeline——核心就是目录结构加 Markdown。agent 用 grep 就能搜索，用文本编辑器就能维护。这种简单性是它能持续运转的关键，也意味着随时可以重构，迁移成本相对可控。

知识工程没有完成的那一天。每一次 agent 和人的协作都在产生新的知识，知识体系本身也在不断演进。重要的不是一开始就设计出完美的体系，而是**先让 agent 开始记，然后让体系在记的过程中自己长出来。**

---

**示例仓库**：`https://github.com/st1page/agent-knowledge-framework` — 从实际使用方式中提取的最小公开示例，包含完整目录结构、两个示例角色（`cli-tool-dev`、`maintainer`）以及通用知识（`principles` / `insights`）。

---

## 附录 A：AGENTS.md 实例

以下是公开示例仓库中的 `AGENTS.md` 全文。

````markdown
# Agent Notes

每个 agent 在开始任何任务前，**必须先通读本文件和角色 `AGENTS.md`**，再开始动手。

## 仓库结构

```
agent-knowledge-framework/
├── base/                       # 通用知识（所有角色共享）
│   ├── experience/             # 不属于特定角色的经验
│   ├── principles/             # 通用原则和规范
│   ├── skills/                 # 通用技能（每个技能目录主入口为 SKILL.md）
│   └── insights/               # 通用洞察（跨角色的规律性认知）
└── roles/                      # 各角色目录
    ├── cli-tool-dev/          # CLI/TUI 工具开发
    ├── maintainer/            # 知识仓库维护
    └── <role>/
        ├── AGENTS.md          # 角色描述 + 知识索引（始终加载）
        ├── principles/        # 角色专有原则
        ├── skills/            # 角色专有技能
        ├── insights/          # 角色专有洞察
        └── experience/        # 角色经验复盘
```

## 知识加载与沉淀

- **常驻 context**：`AGENTS.md`（本文件）+ `roles/<role>/AGENTS.md`
- **按需加载（分层）**：角色/目录 `AGENTS.md` 索引 → skill/principle/insight → experience（按 `source` 或 "Escalate to experience if" 升级）
- **症状/高风险直达**：出现特定报错或做迁移/发布/权限操作时，可直接用关键词检索 experience
- **Context 压缩后**：如果你不确定索引内容或协作规则细节，说明 context 已被压缩，立即重新 `Read` 本文件 + 角色 `AGENTS.md`

详细的加载策略（常驻 vs 按需判断标准、experience 唤醒、摘要写法）见 **`base/principles/knowledge-loading.md`**。

完成重要工作后（PR review 修复、跨层 bug 修复、首次跑通流程、踩坑），agent 应主动沉淀经验。知识分类（experience/skill/principle/insight）、存放位置判断、提炼流程和反模式，见 **`base/knowledge-sedimentation.md`**。

## 多 Agent 协作规则

本仓库假定**多个 coding agent 可能同时修改**，所有写操作必须遵守以下规则。

Agent 工作中涉及两类仓库，规则同样适用：

- **Agent 目录**（本仓库）：存放角色知识、经验、技能文档
- **工作目录**（实际代码仓库）

### 1. 所有写操作必须在 worktree 中进行，禁止直接 push main

**始终创建 worktree**，不要在主工作目录中 `git checkout` 或 `git checkout -b` 切分支。只读操作可以在 main 上进行。

**常见违规模式（禁止）**：
- `git checkout -b <branch>` 然后直接在主工作目录编辑——这不是 worktree，只是切了分支
- "先写完再搬到 worktree"——一旦开始编辑就很难搬，必须一开始就在 worktree 里

**正确流程**：
```bash
git fetch origin
git worktree add .claude/worktrees/<topic> -b <agent>/<topic> origin/main
cd .claude/worktrees/<topic>
# ... 编辑、commit ...
git push -u origin <agent>/<topic>
```

**关键**：永远基于 `origin/main` 创建 worktree，不要基于本地 `main`（可能过时）。详见 `base/principles/git-worktree.md`。

> **检查点**：执行任何 `git checkout -b` 或文件编辑前，先问自己"我现在在 worktree 里吗？"。如果 `pwd` 不包含 `.claude/worktrees/`，停下来，先创建 worktree。

### 2. 通过 PR 合并，agent 不要自行 merge

- push 分支后创建 PR，交给**人工 review 后合并**
- **agent 不得自行 merge PR**，即使有权限也不要执行
- push 时发现冲突：`git fetch origin && git rebase origin/main` 解决后再 push

### 3. push 后同步更新 PR title/description

向已有 PR 的分支 push 新 commit 后，**必须更新 PR title 和 description**，使其反映分支的完整变更。

### 4. 完成后清理 worktree

```bash
git worktree remove .claude/worktrees/<topic>
git branch -d <agent>/<topic>
```

## 开工 Preflight（每个任务都要做）

> 把"开工检查"从思考题变成流程题，避免 context compaction / 口述路径 / 记忆偏差造成的漂移。

### 1）进入角色（硬性）

每个任务的**第一条回复**必须明确：`当前角色` + `已读取 roles/<role>/AGENTS.md`。续接长 session 或不确定流程细节时，按 compaction 处理：重新 `Read` 本文件 + 角色 `AGENTS.md` 后再执行。

### 2）路径必须先验证，禁止猜

**先用 `ls/rg` 找到真实路径，再打开/引用**。找不到就明确说明"仓库内不存在该路径"并给出最接近候选项。常见笔误：`skill` vs `skills`、`AGENTS` vs `SKILL`。

### 3）触发型 skill：要做事先读（硬性）

- 接触凭证 / 权限 / token → 先读 `base/principles/credential-safety.md`
- 长任务（>30min / 需归档）→ 相关原则

### 4）确认"本地=最新"

**触发时机**：开工前、push/提 PR 前、遇到奇怪冲突/缺文件时。对 Agent 目录和工作目录都要检查。

```bash
git fetch origin
git rev-list --left-right --count main...origin/main   # 0 0 = 一致; 0 N = 落后需同步
git pull --rebase origin main                             # 工作区干净时执行
```

> 写操作仍以 `origin/main` 创建 worktree 为准，不要依赖本地 main。
````

## 附录 B：经验沉淀规范（knowledge-sedimentation.md）

以下是公开示例仓库中的 `base/knowledge-sedimentation.md` 全文。

````markdown
# 经验沉淀规范

当用户要求「总结经验」「沉淀一下」时，不要无脑堆到一个文件里。**先判断这条知识的性质和受众，再决定放在哪里**。

## 知识分类与存放位置

| 类别 | 定义 | 存放位置 | 示例 |
|---|---|---|---|
| **experience** | 具体事件的复盘和洞察：踩过的坑、发现的有效做法、关键决策的上下文 | `roles/<role>/experience/` | 「改返回类型后遗漏了下游消费者的适配」「发布流程首次跑通的完整记录」 |
| **skill** | 可复用的操作流程：怎么做某类事、checklist、代码模式 | `roles/<role>/skills/` 或 `base/skills/` | 「跨层 API 重构的分层修改顺序」「FFI 方法接受多种类型的模式」 |
| **principle** | 抽象的行为准则：应该/不应该做什么，与具体技术无关 | `roles/<role>/principles/` 或 `base/principles/` | 「每个功能用独立 worktree」「集成测试必须跑」 |
| **insight** | 从多次 experience 中归纳出的规律性认知：不是操作步骤（skill），也不是行为准则（principle），而是对「为什么会这样」的理解 | `roles/<role>/insights/` 或 `base/insights/` | 「迭代式 review 比一次性修复更可靠」「Rust 做状态维护 + Python 做边界过滤的职责划分模式」 |

## 判断放 role 还是 base

- **只对本角色有意义** → `roles/<role>/`。例如某项目的 FFI 双层提取模式，只有该角色会用到
- **多个角色都可能遇到** → `base/`。例如「改了函数返回类型后要追踪所有消费者」「skip ≠ pass」，这些是通用工程经验

问自己：「如果另一个角色遇到类似情况，这条知识对他有用吗？」有用就放 base。

## 从 experience 提炼到 skill/principle/insight

experience 是原始素材，skill、principle 和 insight 是提炼后的可复用知识。沉淀时要做两步：

1. **先写 experience**：记录具体事件——不只是犯错，也包括有效的做法、关键决策的权衡过程、首次跑通某个流程的完整记录
2. **再看能否提炼**：从 experience 中抽取可复用的模式，补充到已有的 skill、principle 或 insight 文档中；如果是全新主题则新建文档

提炼方向的区分：
- **skill**：「下次遇到同类事该怎么操作？」→ 操作流程、checklist、代码模式
- **principle**：「下次遇到同类事该遵守什么准则？」→ 行为约束、设计原则
- **insight**：「为什么会这样？背后的规律是什么？」→ 规律性认知、架构模式、跨场景的泛化理解

**提炼不求大而全。** skill、principle、insight 都是 role private 的知识，扎根在角色的具体场景里就够了，不需要上升到普适真理。过度抽象反而丢失实用性——「改 API 返回类型后要追踪所有消费者」谁都知道，但「在多层架构下，改返回类型时最外层的间接消费者最容易遗漏」才是有价值的 insight。

不要只写 experience 不提炼——那只是日记。也不要只写 skill 不留 experience——丢失了具体上下文，后人无法理解「为什么有这条规则」和「当时是怎么做到的」。

## Experience 的唤醒（加载）策略：分层加载 + 少量直达

agent 的上下文窗口有限，experience 往往比 skill/principle/insight 更长、更语境化，所以默认不应把 experience 当成"第一入口"。推荐采用 **分层加载（layered retrieval）**：

1. **先从索引入手**：读 `roles/<role>/AGENTS.md` 的知识索引，定位可能相关的条目（skill/principle/insight/experience）。
2. **优先加载提炼知识**：先 `Read` skill/principle/insight（它们更短、更可执行，且包含 triggers）。
3. **再按条件升级到 experience**：当需要"证据/边界/反例/上下文"时，再继续加载 experience。

同时保留"少量直达"的例外：当出现明确的 **症状型线索**（特定报错/异常现象）或正在执行 **高风险任务**（数据迁移、发布/回滚、权限/凭证相关操作等）时，可以直接通过 triggers/关键词去命中某条 experience。

### 让 experience 能被正确唤醒：两处信号

- **在 skill/principle/insight 里写升级条件**：正文里加一个固定小节（推荐命名 `## Escalate to experience if`），列出"什么时候必须回溯 experience"（例如：遇到某类报错、需要做权衡、需要确认边界条件）。
- **experience 自身提供可检索线索（可选）**：experience 文件可在开头增加 `## Triggers（可选）` 小节，写 3-8 个真实关键词（报错片段、组件名、命令名、文件路径、API 名）。这不是为了"常驻加载"，而是为了在症状出现时能被 `rg` 快速命中。

## Questions：记录已知的未知（known unknowns）

四种知识类型（experience/skill/principle/insight）都是**某种程度上已确认的**知识。但工作中经常产生"注意到了、有假说、但没条件验证"的疑问——它不是 experience（还没发生过），不是 skill（不确定能不能这么做），也不适合勉强写成低信心的 insight。

`questions.md` 就是放这些东西的地方。它不是第五种 knowledge type，而是知识的**前体**——一个 question 最终要么被验证变成 experience → skill/principle/insight，要么长期保持 open 状态作为"我们知道自己不知道"的标记。

### 定位

- **不是 TODO**：question 没有 deadline，不要求"尽快解决"。有些疑问可能几个月甚至更久都没有验证机会，这是正常的。
- **不是垃圾场**：写 question 时要带足够的上下文（来源 session、背景、验证思路），让未来遇到相关场景的 agent 能判断"这个 question 和我当前的任务有关吗"。
- **有索引**：在角色 `AGENTS.md` 索引的 Questions 小节标注条目数，让 agent 知道有待验证的问题存在。

### 生命周期

```
session 中产生疑问 → 写入 questions.md（checkbox / ## 小节）
                         ↓ 未来某次 session 有机会验证
                     写 experience → 提炼 skill/principle/insight
                         ↓
                     回到 questions.md 勾 checkbox，注明去向
```

如果一个 question 长期 open，**不要因此删除它**。它的存在本身就是有价值的信号——下次有人碰到相关场景时，看到这个 question 会知道"这里有不确定性，小心"。

### 什么时候写 question

- session 中注意到某个行为"不确定是不是总是这样"但没时间深究
- 提炼 skill/principle 时信心不足（只有一个数据点），先记为 question
- 读文档/代码时发现矛盾或模糊之处，但当前任务不需要解决它

### 格式

存放在 `roles/<role>/questions.md`，模板见 `roles/_template/questions.md`。

## 自动沉淀：完成重要工作后主动反思

**不要等用户提醒才沉淀经验。** 完成以下类型的工作后，agent 应主动提出沉淀：

- PR review 修复（尤其是多轮迭代的）
- 跨层 bug 修复
- 首次跑通某个流程（发布、部署、新工具集成）
- 踩了非显而易见的坑

**具体做法**：工作完成后，主动问自己五个问题：
1. 这次有没有踩坑或发现反直觉的行为？→ 写 experience
2. 能不能提炼出可复用的 checklist 或模式？→ 写/更新 skill
3. 有没有值得记住的抽象原则？→ 写/更新 principle
4. 有没有跨场景可泛化的规律性认知？→ 写/更新 insight
5. 有没有注意到但没条件验证的疑问？→ 写入 questions.md

有内容就创建分支、写文档、提 PR。不需要用户说「沉淀一下」。

## 实操 Checklist（把沉淀当成"可复用产物"交付）

沉淀不是"写点总结"，而是产出让后人**可回溯、可执行、可检索**的知识：

- **证据先行**：命令/日志/样本量/时间范围/异常占比先写清楚，结论能复核。
- **可回溯引用（优先写"产出物"）**：优先补齐本次产出的 PR / commit / 文档（wiki、runbook、设计稿）引用；避免把"机器名/IP/路径"这类上下文硬塞到 References（它们更适合放在「背景/证据」里）。
- **写清决策依据**：关键决策不要只写"我觉得/项目惯例"，要落到「遵循了哪条 principle/insight』或「参考了哪条 experience/skill」，让后续维护者能复用你的判断逻辑。
- **三件套拆分**：experience（发生了什么+证据链）→ skill（下次怎么做）→ insight（为什么会这样）。
- **入口优先**：skill 首步先确认"问题是否存在/范围多大"（bench/基线/smoke/异常占比），再分层排查。
- **前置条件**：写清在哪台机器做、需要哪些权限、哪些步骤无权限就跳过；强调只读安全边界。
- **可检验判据**：每一步给可比较指标（p95、错误率、delta%、抖动幅度）。
- **可检索**：AGENTS.md 摘要与 triggers 覆盖真实关键词；PR 描述随 push 同步更新（自动化脚本避免 shell 误执行文本）。
- **联动更新（新建/修改文件后）**：
  - skill/principle/insight 的 front matter 必须有 `description` + `triggers`（对照本文 Front Matter 规范，不要只抄已有文件——已有文件可能本身不合规）
  - 更新角色 `AGENTS.md` 索引（新增条目、更新 questions 条目数）
  - 交叉引用：experience ↔ skill ↔ plan 之间的链接是否完整且不冗余
  - 通读改动区域：逐块修改后整体回看，检查是否有矛盾或重复

## Experience 写作建议（引用就地）

experience 不要求 YAML front matter，也**不强制**固定章节。重点是保证 **可复核**（证据）、**可回溯**（产出）、**可复用**（决策依据），并把链接/引用写在离它最近的位置：

- **产出物（PR/commit/doc）**：写在「结果」里（或紧挨着对应结论），不要为了凑结构单独加 References。
- **决策依据（principle/insight/skill/experience）**：写在对应的「关键决策」条目里，让读者能复用你的判断逻辑。
- **环境上下文（机器/IP/路径/参数）**：放在「背景/证据」里；这类信息不是"引用"，不要塞进 References 充数。

一个可参考的最小结构：

```markdown
# <一句话标题>

日期: YYYY-MM-DD

## Triggers（可选）
- "能让 `rg` 命中的真实关键词（报错/组件名/命令/路径）"

## 背景
发生在什么仓库/系统/环境？影响范围是什么？

## 证据与现象
贴命令/日志片段/样本范围/统计口径（不要只给结论）。

## 关键决策（含依据）
- 决策：做了什么选择？
  - 备选：还考虑过什么？为什么没选？
  - 依据：引用具体文档路径（principle/insight/skill/experience）或可验证证据

## 结果
最终怎么收敛？修复点/验证方式是什么？
- 产出（若有）：PR/commit/doc 链接或标识（优先写产出物）
- 沉淀（若有）：这次新增/更新了哪些 `skill/principle/insight`（写路径）
```

### 写产出引用的底线（避免"找不到原始上下文"）

- 能贴 URL 就贴 URL（PR 页面、commit 页面、wiki 页面等）。
- 如果暂时还没产出 PR/文档：不要为了"看起来完整"硬写引用；可以先只写 `repo + branch + commit sha`，等产出物出现后再回填。

## Comment 已有 experience

agent 在工作中如果发现已有 experience 失效、需要补充、或与自己的经历相关，应在该 experience 文件末尾添加 comment。

### 触发场景

- **experience 失效**：系统升级、配置变更导致原有经验不再成立
- **跨角色关联**：另一个角色遇到了类似情况但结论不同

### 格式

在 experience 文件末尾添加 `## Comments` section（如果还没有），然后追加：

```markdown
## Comments

### <角色名> (<日期>)
简要说明（1-2 句），链接到相关文档。
```

### 边界

comment 是批注和索引，不是正文。如果 comment 内容开始包含深度分析或规律性归纳，应将其提炼为独立的知识产出（insight、skill 等），comment 里只留链接。

## 更新已有文档 vs 新建文档

- 如果已有文档覆盖了同一主题，**优先在已有文档中追加 section**，而不是新建文件
- 新建文件仅当主题完全独立时使用

## Front Matter 规范

skill、principle、insight 文件必须包含 YAML front matter，将 metadata 与正文分离。agent 通过 front matter 构建索引，决定是否加载全文。

```yaml
---
description: "1-2 句话描述，用于 AGENTS 索引"
triggers:
  - "触发加载的场景关键词"
source:
  - "roles/<role>/experience/xxx.md"
---
```

- **description**（必填）：简明描述这条知识是什么。直接复制到角色 `AGENTS.md` 的知识索引中
- **triggers**（必填）：列出应该加载这条知识的场景关键词。agent 匹配当前任务时用
- **source**（可选）：指向提炼出这条知识的 experience 文件路径。建立从提炼知识到原始素材的可追溯链

建议在正文里补一个固定小节（强烈推荐）：

- `## Escalate to experience if`：列出需要继续加载 experience 的条件（用于分层唤醒，避免"一上来就读长文档"或"永远不回溯导致踩坑复现"）。

示例：

```yaml
---
description: "四层接口（core → facade → binding → wrapper）改签名时的全链路同步流程和常见陷阱"
triggers:
  - "改公开方法签名"
  - "跨层重构"
  - "类型桩同步"
  - "re-export 检查"
source:
  - "roles/<role>/experience/YYYY-MM-DD-example.md"
---
```

experience 文件不需要 front matter——它的 metadata 已经由文件名中的日期和标题承载。

## 反模式

| 反模式 | 问题 | 正确做法 |
|---|---|---|
| 所有经验都堆到一个 experience 文件 | 后续检索困难，不同主题混杂 | 按主题拆分，一个事件一个文件 |
| 只写 experience 不提炼 skill | 知识停留在「那次踩了个坑」，不能指导未来行动 | experience 写完后审视：能否提炼出可复用的 checklist 或模式？ |
| 角色专有知识放到 base | 污染共享知识库，其他角色看了困惑 | 问「其他角色会用到吗？」不会就放 role 下 |
| 通用经验只放在 role 下 | 其他角色重复踩坑 | 通用工程经验（如 git、测试策略、CI）放 base |
````
