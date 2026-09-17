# Agent Note: Remote skill providers and the execution world

Status: proposed

[English](2026-09-17-remote-skill-providers-and-execution-world.md) | 中文

## Problem

skill 能力已经抽象了 skill 正文的来源。`SkillProvider.list()` 与 `get()` 都是异步的，`locator` 是提供方私有的不透明状态，`SkillResourceBase` 在 `directory` 之外还接受 `url` 与 `opaque`。由 HTTP、数据库或对象存储支撑的提供方无需改动 `ctx.skills`、`dsh-tool-skill` 或 agent loop。

没有被抽象的是两条独立轴线之间的关系。skill 存放在哪里由提供方决定；调用方 agent 的 `read`、`bash`、`edit` 工具作用于哪个世界，则由 `ctx.fs` 与 `ctx.shell` 决定。`dsh-fs-local`、`dsh-fs-e2b` 与 `dsh-fs-sandbox` 已经让第二条轴线可替换，`dsh-skill-filesystem` 也已经通过 `ctx.fs` 探测 `.git`，使沙箱工作区在自己的世界里被发现。两条轴线各自都已解决。

缺口在两者的交叉点上。`dsh-skill-filesystem` 产出 `resourceBase: { kind: 'directory', path: locator.directory }`，这是一个裸路径字符串，模型据此解析相对引用，再用 `read` 工具读取。只有当持有 skill 的世界与 `ctx.fs` 指向的世界是同一个世界时，它才成立。一个通过 HTTP 向 `ctx.fs` 为本地的 agent 提供 skill 的团队注册表，以及一个驱动 `ctx.fs` 为 E2B 沙箱的 agent 的本地 skill 目录，交给模型的都是一条解析不到任何东西的路径。`SKILL.md` 之下的渐进式披露会无声失效：目录里仍然列着这个 skill，`skill` 工具仍然返回正文，只有第三层——`references/`、`scripts/`、`assets/`——失败，而此时模型已经开始执行任务。

资源类别所需的东西也不同，单个 `resourceBase` 无法表达。参考文档需要把字节送进模型上下文，不关心是哪个世界产出的。脚本需要存在于 `ctx.shell` 运行的那个世界里，因为模型被告知要执行它。把两者当作同一个问题，就只能在两种做法之间二选一：全部物化，为可能永远不会被读取的文档付出传输代价；或全部远程提供，让脚本无法运行。

远程提供方还处在热路径上。`dsh-tool-skill` 在每个符合条件的 `agent/pre-step` 都调用 `ctx.skills.snapshot()`，因此一个每次 `list()` 都访问网络的提供方，会在每次模型请求前增加一个往返，而不是每个会话一次。提供方是顺序查询的，因此一个慢提供方会拖延其后注册的每个提供方。

## Proposal

保持注册表与消费方不变。新增一个提供方包和一个模型可见的工具，并让两类资源走不同的传输方式。

**参考文档与素材由提供方提供。** `skill_resource({ name, path })` 工具通过持有 skill 正文的同一个提供方读取单个资源，把文本作为工具结果返回。提供方在内部强制路径包含关系、认证与缓存，因此不牵涉任何文件系统世界，`../` 逃逸在源头就被拒绝，而不是交给恰好挂载的那个世界的文件系统策略。

**脚本物化到执行世界。** 提供可执行资源的提供方，在首次使用时通过 `ctx.fs` 把它们暂存到工作区内的目录，并在资源指引中报告该目录的路径。只有脚本被暂存，因此未读取的参考文档不产生任何代价。

**这类提供方的 `resourceBase` 采用 `opaque`。** `opaque` 这一 kind 不承诺任何寻址语义，只承诺提供方会说明如何获取资源。其描述文本同时说明两种传输方式：

```
Resources for this skill are served by provider "team-registry".
Read reference files with the `skill_resource` tool: skill_resource({ name: "…", path: "references/…" }).
Executable scripts are staged into the workspace at /workspace/.skill-cache/<name>/ on first use.
Load referenced resources only as needed.
```

**远程发现从内存中应答。** 远程提供方的 `list()` 返回缓存的候选列表并在后台刷新，当刷新观察到变化时调用 [skill 目录热刷新](../../implemented/feature/2026-07-27-skill-catalog-hot-refresh.zh.md)中那个注册作用域的 `invalidate()`。`list()` 不得在 pre-step 路径上等待网络。刷新失败返回 `{ candidates: lastGood, complete: false }`，注册表已经能处理这种情况：候选仍可加载，聚合快照不可缓存，`dsh-tool-skill` 不发布目录消息并保留模型的 last-good 视图。

### 两条独立轴线

| 轴线 | 问题 | seam | 实现 |
|---|---|---|---|
| skill 存储 | `SKILL.md` 从哪里读出 | `SkillProvider` | `dsh-skill-filesystem`、`dsh-skill-badge` |
| 执行世界 | 模型的文件与 shell 工具作用于何处 | `ctx.fs`、`ctx.shell` | `dsh-fs-local`、`dsh-fs-e2b`、`dsh-fs-sandbox` |

四种组合都是真实部署：本地存储加本地执行是已发布的 CLI；本地存储加沙箱执行是针对 E2B 的本地编写；远程存储加本地执行是团队注册表；远程存储加远程执行是托管 agent。

### 资源类别及其目标世界

| 类别 | 所需 | 传输方式 | 未使用时的代价 |
|---|---|---|---|
| `references/*.md` | 字节进入模型上下文 | `skill_resource` 工具 | 无 |
| `assets/*` | 字节进入模型上下文 | `skill_resource` 工具 | 无 |
| `scripts/*` | 在 `ctx.shell` 世界中可执行 | 通过 `ctx.fs` 暂存 | 首次使用时暂存 |

### 团队共享 skill 注册表

远程存储加本地执行这一组合是第一个消费方，无需任何核心改动。

`dsh-skill-http` 提供方包在 `apply()` 期间同步注册，在其被 await 的 `list()` 内部完成认证，凭据取自 `ctx.credentials` 而非读取环境变量。其 `locator` 携带 `{ id, version }`，使 `get()` 加载确切修订版，陈旧的选择可被检出。它以 rank `150` 注册：低于 `project-dsh`（100），因此项目自有的 skill 始终胜出；高于 `runtime`（250）与用户根目录，因此共享库覆盖个人副本。由于 rank 只在同一层内决定重名，挂载在某个 agent preset 组合中的注册表对其他 preset 不可见。

发布形态是一个 skill bundle 仓库加一个服务端，后者提供目录端点与正文端点。目录响应为每个 skill 携带 `name`、`description`、`version` 与调用策略；正文端点按 id 与版本返回单个 skill 的 Markdown。`skill_resource` 从第三个端点读取，该端点的作用域限定为单个 skill id。

新鲜度由提供方负责，而非注册表——注册表没有 TTL，也无法推断任意远程源何时变化。按及时性从高到低：发布时调用 `invalidate()` 的 webhook 或 server-sent event 流；对目录端点的 ETag 轮询；固定间隔刷新。轮询定时器由 `control.signal` 清理，而 `invalidate()` 在该次注册被释放后是空操作，因此被替换的注册的迟到响应无法污染其后继。

## Alternatives considered

**双世界 `resourceBase`。** 为 `SkillResourceBase` 增加世界标识，并让 `read` 工具据此路由。这会把多世界寻址推进文件系统 seam 和每个文件工具，只为服务一个消费方。`ctx.fs` 刻意为每个 agent 呈现单一世界；skill 子系统不构成打破它的正当理由。

**把每个资源都作为 URL 提供。** 设置 `resourceBase: { kind: 'url' }`，让模型用 web 工具获取资源。这不增加任何机制，但 web 工具不携带注册表凭据，私有库因此被排除在外，而且获取到的脚本仍然无法在 agent 所处的世界中执行。对公开只读库可行，作为通用答案则不够。

**物化整个 skill bundle。** 把每个资源都暂存到执行世界并保留 `kind: 'directory'`。模型不需要新工具，这是它的主要吸引力。但它为通常永远不会被读取的参考文档付出传输与存储代价，需要淘汰机制与按会话隔离，且对素材密集的 skill 扩展性很差。暂存仅保留给脚本，因为那里没有别的办法。

**让注册表缓存远程目录。** 把重试与新鲜度移入 `ctx.skills`，使每个远程提供方都继承它们。注册表无从知晓任意后端因何失效，这正是 `invalidate()` 由提供方驱动的原因；注册表层面的 TTL 对 webhook 支撑的注册表和慢轮询都会猜错。现有的 `complete: false` 观察结果已经能在不依赖缓存的前提下承载瞬时失败。

**随正文一并预取资源。** 在 `skill` 工具结果中返回 `references/` 的内容。这会把第三层披露压回第二层，而这恰恰是工具结果裁剪器要截断的对象：正文加参考文档超过 `thresholdChars` 后会被裁成头尾两段，中间部分被无声丢弃。

## Acceptance criteria

- 通过 HTTP 提供 skill 的提供方能贡献目录、加载正文并提供参考文档，无需改动 `ctx.skills`、`dsh-tool-skill`、`dsh-fs` 或 `agent-loop`。
- 已预热的远程提供方，其 `list()` 无需网络往返即可完成；快照测试固定 pre-step 不产生请求这一事实。
- 注册表失败时，先前发布的目录对模型仍然可见，且不发布替换目录消息。
- `skill_resource` 拒绝逃逸出其 skill 资源根的路径，并以不同的错误拒绝 `invocation.modelInvocable` 为 false 的 skill。
- 暂存到 E2B 执行世界的脚本可在同一会话中由 `bash` 执行；暂存目录在会话结束时被移除。
- 发布了两个版本的 skill，加载候选 `locator` 所指的版本；正文名称与其候选不再匹配时使该提供方失效。

## Risks

- **第二个资源工具在其可见的每次请求中都消耗 schema token。** 目录本身已经按 skill 数量支付描述 token；这为挂载了远程提供方的会话再增加一笔固定 schema 开销。仅在组合了远程提供方处挂载它，可使纯本地部署不受影响。
- **模型必须学会第二条读取路径。** 资源指引是散文，模型仍可能对只做过暂存的路径尝试 `read`。指引显式点名了工具，但没有任何机制强制这一选择；失败的 `read` 是可恢复的错误，而非无声的错误答案。
- **暂存会写入用户工作区。** 执行世界中会出现 `.skill-cache` 目录，可能被误提交。它应当位于 harness 拥有的路径下，且会话结束时的移除是尽力而为：进程被杀死会留下残留。
- **顺序查询提供方使一个慢远程提供方成为所有人的问题。** 后台刷新使其脱离 pre-step 路径，但忽视这一点而等待网络的提供方会拖延其后注册的每个提供方。这是现有注册表的性质，由远程提供方暴露而非引入。
- **缓存目录与已加载正文之间的版本偏移。** 后台刷新可能在模型读到摘要与调用 `skill` 之间替换目录。注册表已经拒绝名称与候选不符的定义；同名下的版本变化则被接受，因此模型可能依据比引导它到此的描述更新的正文行事。
