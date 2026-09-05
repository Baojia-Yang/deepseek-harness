# DeepSeek Harness 有什么不同：从插件组合到会话恢复

## 概述

用过 Claude Code 的人，对 Agent 的工作方式并不陌生：提出任务，模型读取文件、调用工具、观察结果，然后继续工作。这背后负责组织模型请求、工具执行和会话状态的软件，就是 Harness。DeepSeek Harness（简称 DSH）把这些机制都纳入插件体系，让开发者能够替换运行机制、组合应用形态，并从持久记录中延续会话。理解它的特点，可以沿着三个概念展开：Cordis、Everything is a plugin，以及 Session Log。

## 目录

- [DSH 把什么交给了开发者](#dsh-%E6%8A%8A%E4%BB%80%E4%B9%88%E4%BA%A4%E7%BB%99%E4%BA%86%E5%BC%80%E5%8F%91%E8%80%85)
- [Cordis：让插件能够协作和替换](#cordis%E8%AE%A9%E6%8F%92%E4%BB%B6%E8%83%BD%E5%A4%9F%E5%8D%8F%E4%BD%9C%E5%92%8C%E6%9B%BF%E6%8D%A2)
- [Everything is a plugin：组合决定应用的样子](#everything-is-a-plugin%E7%BB%84%E5%90%88%E5%86%B3%E5%AE%9A%E5%BA%94%E7%94%A8%E7%9A%84%E6%A0%B7%E5%AD%90)
- [Session Log：从记录生成上下文和状态](#session-log%E4%BB%8E%E8%AE%B0%E5%BD%95%E7%94%9F%E6%88%90%E4%B8%8A%E4%B8%8B%E6%96%87%E5%92%8C%E7%8A%B6%E6%80%81)
- [什么时候这些差异有价值](#%E4%BB%80%E4%B9%88%E6%97%B6%E5%80%99%E8%BF%99%E4%BA%9B%E5%B7%AE%E5%BC%82%E6%9C%89%E4%BB%B7%E5%80%BC)
- [延伸阅读](#%E5%BB%B6%E4%BC%B8%E9%98%85%E8%AF%BB)

---

## DSH 把什么交给了开发者

假设你想定制一个 Coding Agent。增加一项 Skill、接入一个外部工具、在执行命令前做检查，都是常见需求。再往下一层，你可能还想改变工具的执行方式，替换会话存储，甚至使用自己的 Agent Loop，决定模型何时继续请求、何时停止。

这些需求涉及不同程度的控制权。Claude Code 的公开插件机制提供 Skills、子 Agent、Hooks、MCP 等扩展，适合把知识、工具和工作流程接入已有产品。DSH 的插件范围进一步覆盖运行机制：模型适配、工具注册与执行、会话服务、Agent Loop，以及 Web 界面，都可以通过插件组合来选择和替换。这里比较的是公开提供给开发者的扩展方式。[Claude Code 插件文档](https://code.claude.com/docs/en/plugins)、[DSH 架构说明](../docs/architecture.md#cordis)

因此，理解 DSH 时，值得问的是三个具体问题：我能改变 Agent 的哪些运行机制？我能用这些能力组合出什么应用？一次工作中发生的事情，又如何成为下一次继续工作的依据？

---

## Cordis：让插件能够协作和替换

DSH 使用 Cordis 管理插件。把 Cordis 理解为负责插件协作和生命周期的框架，就足以读懂后面的内容。它最关键的作用有两个：让插件找到需要的能力，以及让插件退出时撤销自己注册的东西。

### 插件通过能力协作

一个工具需要读文件，可以依赖文件服务；需要调用模型，可以依赖模型服务。Cordis 负责把需要服务的插件与提供服务的插件连接起来。插件之间也可以通过事件协作，例如在工具执行前做权限检查，在模型请求前加入上下文。

这让开发者有两种改造方式。希望调整某个环节时，可以在已有扩展点加入插件；希望改变整套机制时，可以提供满足相应接口的替代实现。Agent Loop 本身也是插件，所以开发者能够拥有它的实现，而工具或策略插件也有各自的扩展位置。[Cordis 入门](../docs/cordis-primer.zh.md)、[DSH 扩展位置](../docs/architecture.md#where-new-behavior-goes)

### 插件可以加载，也可以退出

插件运行时会注册工具、服务或事件监听器。Cordis 将这些注册与插件实例的生命周期关联起来：卸载插件时，相应注册也被撤销；依赖某项服务的插件，则随服务的可用性激活或退出。这样，调整插件组合才有明确的生效和清理机制。

![工具插件加载时注册能力，卸载时撤销注册；Agent 根据当前工具注册表获得可用工具](assets/deepseek-harness-differences/01-plugin-lifecycle.svg)图 1：以工具为例，插件的生命周期包括它对运行时产生的注册效果。

这也是“运行时重组”的基础。但需要区分框架能力和应用默认行为：当前内置 Web Profile 支持配置热重载，ACP、SDK、SDK Minimal 等 Profile 在启动时应用配置；插件代码的模块热更新需要另行启用。热重载也不等于任意组件升级都能保留正在进行的工作。[Profile 加载策略](../docs/architecture.md#profiles-and-bundles)

---

## Everything is a plugin：组合决定应用的样子

Everything is a plugin 的含义，可以先从 DSH Web 的设置界面看起。下面这张截图里，工具、模型服务和 Agent 的运行机制，都出现在同一份插件清单中。

### 从设置界面看见插件

<img src="./deepseek-harness-differences.assets/web-plugin-settings.webp" alt="DSH Web 插件设置界面：agent-loop 与 fs-sandbox 被红框标出，tool-todo、tool-goal 和 tool-web 等显示“预设中启用”" width="543" />图 2：DSH Web 的真实插件清单。截图及红框标注由作者提供。

红框里的 `agent-loop` 负责驱动 Agent 的执行循环，`fs-sandbox` 负责带沙箱策略的文件访问。同一屏中的 `tools` 提供工具注册与执行机制，`system-prompt` 组织系统提示，`llm-deepseek` 接入模型。这些名称说明，DSH 的“插件”既包括模型能调用的工具，也包括让 Agent 运转起来的基础机制。

截图还显示了两种配置状态。“已启用”表示该条目在应用的全局配置中启用；“预设中启用”表示全局条目没有直接启用，而是有 Agent Preset 提供这项插件，例如 `tool-todo` 和 `tool-web`。这里的 Preset 可以先理解为供会话选择的一组 Agent 能力。这个标签不表示所有会话都有这项工具，也不表示对应预设此刻一定已经运行。[插件清单界面](../packages/client/ui-settings-plugin-inventory/README.md)

### 从插件清单映射到 Plugin Tree

设置页便于逐项查看插件。Plugin Tree 则进一步展示插件挂在哪里、哪些插件组成一棵子树。下面保留 Web 配置中的关键节点，并展开 `standard` 预设的一小部分，让截图中的名字有一个具体位置。

![Web 插件树的关键节点：根下包含 agent-loop、fs-sandbox、tools、system-prompt、llm-deepseek 及 Web 应用插件；agent-presets 管理的 standard 子树中包含 tool-todo、tool-goal 和 tool-web](assets/deepseek-harness-differences/06-web-plugin-tree.svg)图 3：按 Web 的真实配置选取节点；名称省略 `@deepseek-ai/dsh-` 前缀。以 `standard` 已被会话使用为例，虚线省略了预设的中间挂载层；背景分区仅用于阅读，不是额外的父插件。

这棵树可以从两个层次理解。应用层提供模型访问、会话持久化、沙箱、工具注册表等公共服务，也提供 Web 服务与界面。预设子树则提供供会话使用的工具、提示词和 Skill。同一个 Web 进程可以同时服务使用不同预设的会话；使用同一预设的会话共享那组插件的装配，各自的会话状态仍然独立。[Web 配置](../packages/bundle/web-app/cordis.patch.yml)、[Standard 预设](../packages/preset/agent-presets/presets/standard/agent.cordis.yml)、[Agent Presets](../packages/preset/agent-presets/README.md)

例如，截图中的 `web-search-deepseek` 提供搜索服务，`tool-web` 把网页能力作为工具提供给模型。前者属于共享服务，后者由预设决定是否提供给会话。于是，“应用具备这项基础能力”和“这个 Agent 能调用这项工具”是两个可以分别配置的问题。

树边表示挂载层次，并不表示调用顺序；`fs-sandbox` 与 `agent-loop` 并列，也不意味着两者互不依赖。服务如何被找到、哪些注册对某个会话可见，由 Cordis 的 Context 和作用域机制管理。读到这里，只需要记住：插件树组织了运行中的能力，而能力的可见范围也可以被控制。

### Web 与 ACP：改变应用面向谁

如果选择另一套插件，应用会怎样变化？先看 `web` 与 `acp` 这两个启动配置，DSH 将这样的命名配置称为 Profile。

Web 面向浏览器中的人，增加 Web 服务、会话界面、设置页和预设选择。ACP 是 Agent Client Protocol；DSH 的 ACP Profile 面向自动化调用方，增加通过标准输入输出通信的 ACP 服务，由其他程序提交任务、接收更新和管理会话。两者复用模型、Agent Loop、工具和持久化等公共插件，但启动的是各自独立的应用。[Web Bundle](../packages/bundle/web-app/README.md)、[ACP Bundle](../packages/bundle/acp-app/README.md)

![web 和 acp 分别包含公共运行能力；web 增加 Web 服务、浏览器界面与 Agent Presets，acp 增加通过标准输入输出通信的 ACP 协议服务](assets/deepseek-harness-differences/02-web-acp.svg)图 4：保留公共运行能力，改变应用入口与会话能力的组织方式。图中按能力分组，不表示插件之间的父子关系。

| 对比项 | `web` | `acp` |
| --- | --- | --- |
| 直接使用者 | 在浏览器里工作的用户 | 实现 ACP 的自动化调用方 |
| 交互方式 | 会话界面、设置页、模型与预设选择 | 协议请求、响应和会话更新；不包含 Web 界面 |
| Agent 能力组织 | 公共服务加预设；会话选择对应能力组合 | 默认使用应用中挂载的工具组合，不包含 Web 的预设管理 |
| 配置生效 | 内置 Profile 支持配置热重载 | 内置 Profile 在启动时应用配置 |

这不只是给同一个界面换一种访问方式。Web 需要的展示、设置与预设管理，ACP 不必全部携带；ACP 需要的协议服务，也可以作为插件接到公共能力上。对开发者来说，可组合的范围覆盖了整个应用形态。

### SDK 与 SDK Minimal：改变模型实际拿到什么

`sdk` 与 `sdk-minimal` 则展示另一种差异：两者都通过 DSH 的 SDK JSON-RPC 协议供程序调用，也都保留模型请求、Agent Loop、Session Log 与持久化，但模型拿到的工具和上下文不同。

`sdk` 提供完整的默认 Agent 组合，包含 Shell、文件操作、搜索、Skill、子 Agent 等工具及配套服务。`sdk-minimal` 独立列出自己需要的插件，默认只向模型提供持久 Shell 和文本编辑两种工具。这里的“精简”指能力组合精简，不是换用了更小的模型，也不是换了一套 SDK 协议。[SDK Bundle](../packages/bundle/sdk-app/README.md)、[SDK Minimal Bundle](../packages/bundle/sdk-minimal/README.md)

![同一种 SDK 协议驱动两种 Agent 组合：sdk 提供丰富工具、上下文与配套服务，sdk-minimal 保留持久 Shell 和文本编辑；两者都有模型调用、Agent Loop 和 Session Log](assets/deepseek-harness-differences/03-sdk-minimal.svg)图 5：调用方式相近，不代表模型面对的工作环境相同。

| 对比项 | `sdk` | `sdk-minimal` |
| --- | --- | --- |
| 默认模型工具 | Shell、文件操作与搜索、Skill、子 Agent、Todo、Web 等 | 持久 Shell 与文本编辑 |
| Shell 形态 | 默认 Shell 工具及后台任务控制工具 | 持久 Shell，在多次调用之间保留状态；不提供单独的后台任务控制工具 |
| 提示上下文 | 系统提示、运行环境信息、工作区指令等上下文贡献 | 简短默认提示，不加入 Harness 身份与运行环境段落，也不组合工作区指令插件 |
| 配套能力 | 包含上下文压缩、Skill 加载、子 Agent、设置等插件 | 默认不组合这些插件 |
| 会话基础 | 模型调用、循环执行、会话日志与持久化 | 同样保留，日志默认保存为未压缩 JSONL |

例如，同样提出“检查项目并修改一个文件”，普通 SDK 的 Agent 可以获得专门的文件与搜索工具，也可以使用预先组合好的 Skill 或子 Agent；Minimal 的 Agent 主要通过 Shell 检查项目，再使用文本编辑工具修改内容。插件组合改变了模型可选择的操作，以及它开始工作时获得的信息。

Minimal 的持久 Shell 在 Linux、macOS 上使用 Bash，在 Windows 上使用 PowerShell。它默认采用 `danger-full-access`，文本编辑直接使用本地文件系统，运行环境的隔离需要由调用方负责。功能少不等于权限小；比较这两种组合时，也要检查它们各自的权限配置。[Minimal 配置](../packages/bundle/sdk-minimal/cordis.patch.yml)

### 插件树是怎样组合出来的

看过这些具体应用，再回头看 Plugin、Bundle 和 Profile，它们的分工就更清楚了：Plugin 提供能力；Bundle 分发一组插件配置及其依赖；Profile 选择本次启动按什么顺序使用哪些 Bundle。Plugin Tree 是这些配置被加载后形成的挂载结构。

前面四个 Profile 的内置组合分别是：

| Profile | 按顺序应用的 Bundle |
| --- | --- |
| `web` | `dsh-base` → `dsh-web-app` |
| `acp` | `dsh-base` → `dsh-acp-app` |
| `sdk` | `dsh-base` → `dsh-sdk-app` |
| `sdk-minimal` | `dsh-sdk-minimal`，独立提供完整配置，不叠加 `dsh-base` |

Bundle 通过 Patch 提供配置。Patch 可以插入插件条目，也可以按条目的标识调整配置或启停状态。组合从空配置开始：先依次应用 Bundle 的 Patch，再应用 Profile 自己的 Patch、Harness Home 中的 Patch，以及本次启动额外指定的 Patch。后面的层可以覆盖前面的选择，最终配置交给 Cordis 加载。

![web Profile 选择 dsh-base 和 dsh-web-app，按顺序应用它们的配置并叠加用户 Patch，再由 Cordis 加载；来自两个 Bundle 的 agent-loop、fs-sandbox、host-webserver 和 agent-presets 都可以成为根下的插件节点](assets/deepseek-harness-differences/04-composition.svg)图 6：Bundle 是配置来源，不必成为树中的父节点。来自不同 Bundle 的插件可以在同一层挂载。

以 Web 为例，`dsh-base` 提供 `agent-loop`、`fs-sandbox` 等条目；`dsh-web-app` 加入 Web 服务、界面和 `agent-presets`，同时调整部分基础条目，把模型工具交由预设提供。因此，Bundle 的组合既能增加插件，也能改变已有插件的配置和启用位置。

Profile 与 Preset 也各有作用：Profile 选择整个应用，Preset 选择会话使用的 Agent 能力。Web 设置页中“已启用”和“预设中启用”的区别，正是这种分层组合在产品中的直接表现。[Profile 与 Bundle](../docs/architecture.md#profiles-and-bundles)、[Agent Presets](../packages/preset/agent-presets/README.md)

---

## Session Log：从记录生成上下文和状态

插件组合回答“这个 Agent 拥有什么”，Session Log 则回答“这个会话发生过什么”。在 DSH 中，Session Log 是持续追加的会话事件序列，记录用户输入、模型响应、工具调用与结果，以及需要保存的会话状态。

### 模型看到的内容，必须能够从日志重建

假设 Agent 读取了一个文件，随后根据文件内容回答问题。模型下一轮请求所使用的对话历史，应当能够根据日志中的输入、工具调用和结果重新生成。额外注入的模型可见上下文，同样需要有日志依据。

DSH 把“模型可见内容必须能够从日志重建”作为明确的架构规则。日志因此同时参与运行和展示：模型历史、用户查看的执行过程，以及由事件记录的 Goal、Todo 等状态，可以从同一条事件流生成各自的视图。[Session Log 架构规则](../docs/architecture.md#session-log)

![同一份 Session Log 分别生成模型上下文、会话执行历史和持久状态，并为恢复与分叉提供记录依据](assets/deepseek-harness-differences/05-session-log.svg)图 7：同一份会话事实，可以服务于模型、用户界面和后续会话操作。

这些视图不必包含完全相同的内容。例如，上下文压缩会改变下一次模型请求使用的历史，但已经发生的会话事实仍保留在日志里。用户可查看的历史与模型当前使用的上下文，可以有各自的呈现方式。

### 恢复会话时，恢复的是什么

一个进程退出后，新进程可以读取已保存的日志，重建会话历史和相关状态，再把新的输入、模型响应和工具结果追加下去。分叉则从选定的历史位置开始形成另一条会话。原有记录成为继续工作的依据。[Session 子系统](../docs/subsystems/session.zh.md)

这里的恢复有明确范围：日志保存的是会话事实。工作区文件可能已经变化，旧的 Shell 进程可能已经结束，外部服务也可能返回不同结果。恢复会话不代表把整个运行环境还原到过去，更不保证模型再次执行时产生完全相同的答案。

### DSH 与 Claude Code 的会话能力

Claude Code 也会持久保存会话，并通过这些记录恢复对话历史、工具调用和结果；它的 SDK 也支持恢复和分叉。两者都能在已有对话的基础上继续工作。[Claude Code 会话管理](https://code.claude.com/docs/en/sessions)、[Claude Agent SDK 会话管理](https://code.claude.com/docs/en/agent-sdk/sessions)

DSH 向开发者开放了这套会话机制的组成和扩展方式：会话事件是模型上下文及多种产品视图的共同依据，开发者能够围绕这些事件增加持久状态和视图，也能替换持久化实现。例如，插件增加一项需要在恢复后保留的会话状态时，可以把变化写成事件，再从历史事件恢复相应状态。插件体系与 Session Log 在这里相连，让新增能力也能够参与会话的记录与延续。

---

## 什么时候这些差异有价值

如果目标是让 Agent 完成日常编码，实际效果仍需要结合模型、工具和任务来判断。本文讨论的架构选择，也不能直接推出某个 Agent 的编码能力更强。

当目标变成构建自己的 Agent 产品时，DSH 的这些能力就更容易体现价值：可以调整运行机制，选择完整或精简的能力组合，为浏览器或自动化程序提供不同入口，并让需要延续的会话事实进入统一记录。Cordis 管理插件如何协作与退出，插件组合决定应用和 Agent 的能力，Session Log 为上下文、状态及后续恢复提供依据。

DSH 当前处于开发者预览阶段。理解这套设计的意义，在于知道哪些部分可以由自己决定，以及每种选择会怎样影响实际运行。[项目状态](../README.md#developer-preview)

---

## 延伸阅读

- [DSH 架构说明](../docs/architecture.zh.md)：插件组合、运行过程和 Session Log 的完整关系。
- [Cordis 入门](../docs/cordis-primer.zh.md)：服务、事件、依赖和插件生命周期的基本机制。
- [SDK Minimal](../packages/bundle/sdk-minimal/README.md)：精简组合提供的工具及其使用条件。
- [Session 子系统](../docs/subsystems/session.zh.md)：会话事件、存储与恢复的进一步说明。

## 开发备注

无。