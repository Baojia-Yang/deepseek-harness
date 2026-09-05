# DeepSeek Harness 有什么不同：从插件组合到会话恢复

## 概述

给 Agent 增加一个工具，和改变 Agent 的运行方式，是一回事吗？

用过 Claude Code 的人，对前一种需求应该不陌生：接入工具、添加 Skill，让 Agent 能处理更多任务。但如果想改的是工具怎样执行、模型何时继续请求，甚至整个会话怎样保存和恢复，问题就进入了 Harness 的范围。Harness 指的正是组织模型请求、工具执行和会话状态的那部分软件。

这也是我更关注 DeepSeek Harness（简称 DSH）的地方：**开发者能定制的，不只有 Agent 使用的工具和知识，还有让 Agent 运转起来的机制。** 本文沿着 Cordis、Everything is a plugin 和 Session Log 三个概念展开，从界面、应用组合和会话恢复这些能直接观察到的能力，看清它与 Claude Code 等 Agent 的差异。代码实现先不深入，重点是这些设计让我们能够做什么，以及不能据此推断什么。

## 目录

- [DSH 把什么交给了开发者](#dsh-%E6%8A%8A%E4%BB%80%E4%B9%88%E4%BA%A4%E7%BB%99%E4%BA%86%E5%BC%80%E5%8F%91%E8%80%85)
- [Cordis：让插件能够协作和替换](#cordis%E8%AE%A9%E6%8F%92%E4%BB%B6%E8%83%BD%E5%A4%9F%E5%8D%8F%E4%BD%9C%E5%92%8C%E6%9B%BF%E6%8D%A2)
- [Everything is a plugin：组合决定应用的样子](#everything-is-a-plugin%E7%BB%84%E5%90%88%E5%86%B3%E5%AE%9A%E5%BA%94%E7%94%A8%E7%9A%84%E6%A0%B7%E5%AD%90)
- [Session Log：从记录生成上下文和状态](#session-log%E4%BB%8E%E8%AE%B0%E5%BD%95%E7%94%9F%E6%88%90%E4%B8%8A%E4%B8%8B%E6%96%87%E5%92%8C%E7%8A%B6%E6%80%81)
- [什么时候这些差异有价值](#%E4%BB%80%E4%B9%88%E6%97%B6%E5%80%99%E8%BF%99%E4%BA%9B%E5%B7%AE%E5%BC%82%E6%9C%89%E4%BB%B7%E5%80%BC)
- [延伸阅读](#%E5%BB%B6%E4%BC%B8%E9%98%85%E8%AF%BB)

---

## DSH 把什么交给了开发者

先把开头的问题说具体一点。假设你想定制一个 Coding Agent：开始只是加一项 Skill、接一个外部工具，或者在执行命令前做检查。再进一步，你希望改变工具的执行方式，使用自己的会话存储，甚至替换 Agent Loop，决定模型何时继续请求、何时停止。这几类需求都叫“定制 Agent”，但需要的控制权并不一样。

Claude Code 的公开插件机制提供 Skills、子 Agent、Hooks、MCP 等扩展，可以把知识、工具和工作流程接入已有产品。DSH 的插件范围则覆盖到运行机制：模型适配、工具注册与执行、会话服务、Agent Loop，连 Web 界面也参与插件组合。开发者能够选择和替换这些部分。这里比较的是两者公开提供给开发者的扩展方式，而不是笼统地判断谁“更开放”。[Claude Code 插件文档](https://code.claude.com/docs/en/plugins)、[DSH 架构说明](../docs/architecture.md#cordis)

所以，读这篇文章时可以一直带着三个问题：Agent 的哪些运行机制可以改？这些能力能组合出什么应用？会话中发生的事情，又怎样成为下一次继续工作的依据？这三个问题，分别对应下面的三部分。

---

## Cordis：让插件能够协作和替换

把各项机制做成插件之后，还需要解决两个问题：插件怎样找到彼此提供的能力？替换或卸载一个插件时，它留下的注册又由谁清理？DSH 使用 Cordis 来管理这些协作和生命周期。理解这两件事，就能看懂它为什么支持运行机制的组合与替换。

### 插件通过能力协作

比如，一个工具需要读文件，可以依赖文件服务；需要调用模型，可以依赖模型服务。Cordis 负责连接使用服务和提供服务的插件。插件之间还可以通过事件协作，例如在工具执行前做权限检查，在模型请求前加入上下文。

对应到开发者手里的选择，就有两种改法：只想调整某个环节，可以在已有扩展点加入插件；想改变整套机制，则可以提供满足相应接口的替代实现。**Agent Loop 本身也是插件，意味着执行循环的实现也可以由开发者决定。** 工具和策略同样有各自的扩展位置，并不是每次定制都需要改 Loop。[Cordis 入门](../docs/cordis-primer.zh.md)、[DSH 扩展位置](../docs/architecture.md#where-new-behavior-goes)

### 插件可以加载，也可以退出

再看一个更直观的问题：工具插件加载时注册了工具，卸载时该怎样处理？如果工具还留在注册表里，插件虽然退出了，它的影响却没有被清理。

Cordis 把工具、服务和事件监听器等注册与插件实例的生命周期关联起来。卸载插件时，相应注册也被撤销；依赖某项服务的插件，则随服务的可用性激活或退出。图中以工具为例，Agent 下一次组装工具列表时，读取的是当时注册表里可用的工具。

![工具插件加载时注册能力，卸载时撤销注册；Agent 根据当前工具注册表获得可用工具](assets/deepseek-harness-differences/01-plugin-lifecycle.svg)*图 1：以工具为例，插件的生命周期包括它对运行时产生的注册效果。*

这就是理解“运行时重组”的关键：变化既能被应用，相应的注册也能随插件退出而撤销。不过，框架支持什么，与具体应用默认开启什么，要分开看。当前内置 Web Profile 支持配置热重载，ACP、SDK、SDK Minimal 等 Profile 在启动时应用配置；插件代码的模块热更新需要另行启用。**配置热重载不等于任意组件都能无损升级**，正在进行的工作能否保留，还要看具体组件。[Profile 加载策略](../docs/architecture.md#profiles-and-bundles)

---

## Everything is a plugin：组合决定应用的样子

Everything is a plugin 听起来很抽象。它到底覆盖了哪些东西？先看 DSH Web 的设置界面：工具、模型服务和 Agent 的运行机制，都出现在同一份插件清单里。

### 从设置界面看见插件

<img src="./deepseek-harness-differences.assets/web-plugin-settings.webp" alt="DSH Web 插件设置界面：agent-loop 与 fs-sandbox 被红框标出，tool-todo、tool-goal 和 tool-web 等显示“预设中启用”" width="543" />*图 2：DSH Web 的真实插件清单。截图及红框标注由作者提供。*

先留意红框里的两个名字：`agent-loop` 驱动 Agent 的执行循环，`fs-sandbox` 负责带沙箱策略的文件访问。再看同一屏的 `tools`、`system-prompt` 和 `llm-deepseek`，它们分别提供工具注册与执行机制、组织系统提示、接入模型。这里列出的，既有模型能调用的工具，也有让 Agent 运转起来的基础机制。

还有一个容易忽略的细节：有些条目显示“已启用”，有些显示“预设中启用”。前者表示该条目在应用的全局配置中启用；后者表示全局条目没有直接启用，而是有 Agent Preset 提供这项插件，比如 `tool-todo` 和 `tool-web`。Preset 可以先理解为供会话选择的一组 Agent 能力。“预设中启用”不代表所有会话都有这项工具，也不表示对应预设此刻一定已经运行。[插件清单界面](../packages/client/ui-settings-plugin-inventory/README.md)

### 从插件清单映射到 Plugin Tree

清单告诉我们有哪些插件，但还没有展示它们挂在哪里、哪些插件属于同一棵子树。把刚才的名字放回 Plugin Tree，关系就更直观了。下图保留 Web 配置的关键节点，并展开 `standard` 预设的一小部分。

![Web 插件树的关键节点：根下包含 agent-loop、fs-sandbox、tools、system-prompt、llm-deepseek 及 Web 应用插件；agent-presets 管理的 standard 子树中包含 tool-todo、tool-goal 和 tool-web](assets/deepseek-harness-differences/06-web-plugin-tree.svg)*图 3：按 Web 的真实配置选取节点；名称省略* `@deepseek-ai/dsh-` *前缀。以* `standard` *已被会话使用为例，虚线省略了预设的中间挂载层；背景分区仅用于阅读，不是额外的父插件。*

先看应用层：模型访问、会话持久化、沙箱、工具注册表等公共服务在这里，Web 服务和界面也在这里。再看预设子树：会话使用的工具、提示词和 Skill 由它提供。同一个 Web 进程可以同时服务使用不同预设的会话；使用同一预设的会话共享那组插件的装配，各自的会话状态仍然独立。[Web 配置](../packages/bundle/web-app/cordis.patch.yml)、[Standard 预设](../packages/preset/agent-presets/presets/standard/agent.cordis.yml)、[Agent Presets](../packages/preset/agent-presets/README.md)

拿网页搜索举个例子。截图中的 `web-search-deepseek` 提供搜索服务，`tool-web` 把网页能力作为工具提供给模型。前者属于共享服务，后者由预设决定是否提供给会话。也就是说，**应用具备一项基础能力，不等于每个会话里的 Agent 都能调用相应工具。** 这两件事可以分别配置。

读图时还要注意：树上的连线表示挂载层次，不是调用顺序。`fs-sandbox` 与 `agent-loop` 并列，也不意味着两者互不依赖。服务如何被找到、哪些注册对某个会话可见，由 Cordis 的 Context 和作用域机制管理。这里不展开这些机制，只保留一个理解：插件树组织运行中的能力，同时允许控制能力的可见范围。

### Web 与 ACP：改变应用面向谁

看清一棵树之后，可以继续问：如果换一种组合，应用会变成什么样？先比较 `web` 和 `acp`。它们是两套命名的启动配置，DSH 把这样的配置称为 Profile。

Web 面向浏览器中的人，所以组合里有 Web 服务、会话界面、设置页和预设选择。ACP 是 Agent Client Protocol；DSH 的 ACP Profile 面向自动化调用方，提供通过标准输入输出通信的 ACP 服务，让其他程序提交任务、接收更新和管理会话。两者复用模型、Agent Loop、工具和持久化等公共插件，但启动的是各自独立的应用。[Web Bundle](../packages/bundle/web-app/README.md)、[ACP Bundle](../packages/bundle/acp-app/README.md)

![web 和 acp 分别包含公共运行能力；web 增加 Web 服务、浏览器界面与 Agent Presets，acp 增加通过标准输入输出通信的 ACP 协议服务](assets/deepseek-harness-differences/02-web-acp.svg)*图 4：保留公共运行能力，改变应用入口与会话能力的组织方式。图中按能力分组，不表示插件之间的父子关系。*

| 对比项 | `web` | `acp` |
| --- | --- | --- |
| 直接使用者 | 在浏览器里工作的用户 | 实现 ACP 的自动化调用方 |
| 交互方式 | 会话界面、设置页、模型与预设选择 | 协议请求、响应和会话更新；不包含 Web 界面 |
| Agent 能力组织 | 公共服务加预设；会话选择对应能力组合 | 默认使用应用中挂载的工具组合，不包含 Web 的预设管理 |
| 配置生效 | 内置 Profile 支持配置热重载 | 内置 Profile 在启动时应用配置 |

我觉得这组对比有意思的地方，在于组合决定了应用面向谁。Web 需要展示、设置和预设管理，ACP 不必全部携带；ACP 需要的协议服务，则作为插件与公共能力组合。开发者选择的不只是一个访问入口，也包括这个应用需要哪些功能。

### SDK 与 SDK Minimal：改变模型实际拿到什么

如果调用方式不变，插件组合还会带来什么差异？`sdk` 与 `sdk-minimal` 正好说明另一件事：应用都可以通过 DSH 的 SDK JSON-RPC 协议供程序调用，都保留模型请求、Agent Loop、Session Log 与持久化，但模型实际拿到的工具和上下文可以很不一样。

`sdk` 提供完整的默认 Agent 组合，包含 Shell、文件操作、搜索、Skill、子 Agent 等工具及配套服务。`sdk-minimal` 则独立列出自己需要的插件，默认只向模型提供持久 Shell 和文本编辑两种工具。这里先明确一点：**Minimal 精简的是能力组合，不是换了更小的模型，也不是另一套 SDK 协议。** [SDK Bundle](../packages/bundle/sdk-app/README.md)、[SDK Minimal Bundle](../packages/bundle/sdk-minimal/README.md)

![同一种 SDK 协议驱动两种 Agent 组合：sdk 提供丰富工具、上下文与配套服务，sdk-minimal 保留持久 Shell 和文本编辑；两者都有模型调用、Agent Loop 和 Session Log](assets/deepseek-harness-differences/03-sdk-minimal.svg)*图 5：调用方式相近，不代表模型面对的工作环境相同。*

| 对比项 | `sdk` | `sdk-minimal` |
| --- | --- | --- |
| 默认模型工具 | Shell、文件操作与搜索、Skill、子 Agent、Todo、Web 等 | 持久 Shell 与文本编辑 |
| Shell 形态 | 默认 Shell 工具及后台任务控制工具 | 持久 Shell，在多次调用之间保留状态；不提供单独的后台任务控制工具 |
| 提示上下文 | 系统提示、运行环境信息、工作区指令等上下文贡献 | 简短默认提示，不加入 Harness 身份与运行环境段落，也不组合工作区指令插件 |
| 配套能力 | 包含上下文压缩、Skill 加载、子 Agent、设置等插件 | 默认不组合这些插件 |
| 会话基础 | 模型调用、循环执行、会话日志与持久化 | 同样保留，日志默认保存为未压缩 JSONL |

把这些差异放进一个任务里更容易理解。同样要求“检查项目并修改一个文件”，普通 SDK 的 Agent 可以使用专门的文件与搜索工具，也可以使用预先组合好的 Skill 或子 Agent；Minimal 的 Agent 主要通过 Shell 检查项目，再用文本编辑工具修改内容。任务没有变，模型可选择的操作、开始工作时获得的信息，却随着插件组合发生了变化。

不过，“工具少”不能直接理解为“权限小”。Minimal 的持久 Shell 在 Linux、macOS 上使用 Bash，在 Windows 上使用 PowerShell，默认采用 `danger-full-access`；文本编辑直接使用本地文件系统，运行环境的隔离需要由调用方负责。选择哪一种组合时，除了看工具和上下文，也要看它们各自的权限配置。[Minimal 配置](../packages/bundle/sdk-minimal/cordis.patch.yml)

### 插件树是怎样组合出来的

前面已经看到了不同组合的结果。接下来再看配置：从选择 `web`，到真正运行起那棵插件树，中间发生了什么？

这里需要区分 Plugin、Bundle 和 Profile。Plugin 提供具体能力；Bundle 分发一组插件配置及其依赖；Profile 决定本次启动按什么顺序使用哪些 Bundle。配置被加载后，才形成运行中的 Plugin Tree，也就是插件的挂载结构。

把前面四个 Profile 放在一起看，区别就很明确：

| Profile | 按顺序应用的 Bundle |
| --- | --- |
| `web` | `dsh-base` → `dsh-web-app` |
| `acp` | `dsh-base` → `dsh-acp-app` |
| `sdk` | `dsh-base` → `dsh-sdk-app` |
| `sdk-minimal` | `dsh-sdk-minimal`，独立提供完整配置，不叠加 `dsh-base` |

那如果想沿用一个 Bundle，但调整其中某个插件呢？这就要用到 Patch。Bundle 本身通过 Patch 提供配置；Patch 既可以插入插件条目，也可以按条目的标识调整配置或启停状态。

组合从空配置开始，先按顺序应用 Bundle 的 Patch，再应用 Profile 自己的 Patch、Harness Home 中的 Patch，以及本次启动额外指定的 Patch。后面的层可以覆盖前面的选择，最终配置交给 Cordis 加载。下图把这个过程和最后的插件树连在了一起。

![web Profile 选择 dsh-base 和 dsh-web-app，按顺序应用它们的配置并叠加用户 Patch，再由 Cordis 加载；来自两个 Bundle 的 agent-loop、fs-sandbox、host-webserver 和 agent-presets 都可以成为根下的插件节点](assets/deepseek-harness-differences/04-composition.svg)*图 6：Bundle 是配置来源，不必成为树中的父节点。来自不同 Bundle 的插件可以在同一层挂载。*

回到 Web 这个例子，`dsh-base` 提供 `agent-loop`、`fs-sandbox` 等条目；`dsh-web-app` 加入 Web 服务、界面和 `agent-presets`，同时调整部分基础条目，把模型工具交由预设提供。所以，组合 Bundle 既可以增加插件，也可以改变已有插件的配置和启用位置。

最后再区分两个名字相近的概念：**Profile 选择整个应用，Preset 选择会话使用的 Agent 能力。** 再回头看 Web 设置页中的“已启用”和“预设中启用”，就能把界面上的状态与背后的组合方式对应起来了。[Profile 与 Bundle](../docs/architecture.md#profiles-and-bundles)、[Agent Presets](../packages/preset/agent-presets/README.md)

---

## Session Log：从记录生成上下文和状态

前两部分讲的是 Agent 由什么组成。接下来看另一个问题：Agent 工作了一段时间，进程退出了，再打开时凭什么接着做？

这就涉及 Session Log。在 DSH 中，它是一条持续追加的会话事件序列，记录用户输入、模型响应、工具调用与结果，以及需要保存的会话状态。理解它的重点，是看这些记录怎样被后续运行使用。

### 模型看到的内容，必须能够从日志重建

假设 Agent 读取了一个文件，随后根据文件内容回答问题。下一轮模型请求使用的对话历史，就需要能够从日志里的输入、工具调用和结果重新生成。如果额外向模型注入了上下文，这些内容同样需要有日志依据。

DSH 把这件事规定为一条架构规则：**模型可见的内容，必须能够从 Session Log 重建。** 因此，日志同时参与运行和展示：模型使用的历史、用户查看的执行过程，以及由事件记录的 Goal、Todo 等状态，都可以从同一条事件流生成各自的视图。[Session Log 架构规则](../docs/architecture.md#session-log)

![同一份 Session Log 分别生成模型上下文、会话执行历史和持久状态，并为恢复与分叉提供记录依据](assets/deepseek-harness-differences/05-session-log.svg)*图 7：同一份会话事实，可以服务于模型、用户界面和后续会话操作。*

同一份日志，为什么还需要不同视图？看上下文压缩就容易理解：压缩会改变下一次模型请求使用的历史，但已经发生的会话事实仍保留在日志里。用户可以查看的历史，与模型这一轮实际使用的上下文，不必完全相同。

### 恢复会话时，恢复的是什么

进程退出后，新进程可以读取已保存的日志，重建会话历史和相关状态，再继续追加新的输入、模型响应和工具结果。分叉则从选定的历史位置开始形成另一条会话。也就是说，原有记录直接成为继续工作的依据。[Session 子系统](../docs/subsystems/session.zh.md)

不过，这里恢复的是会话，不是整个运行环境。工作区文件可能已经变化，旧的 Shell 进程可能已经结束，外部服务也可能返回不同结果。日志保存了会话事实，不代表这些外部条件都能还原到过去，更不保证模型再次执行时给出完全相同的答案。

### DSH 与 Claude Code 的会话能力

那么，这与 Claude Code 的会话恢复有什么不同？先看两者共有的能力：Claude Code 也会持久保存会话，并通过这些记录恢复对话历史、工具调用和结果；它的 SDK 也支持恢复和分叉。两者都能在已有对话的基础上继续工作，不能仅凭“能恢复会话”来区分它们。[Claude Code 会话管理](https://code.claude.com/docs/en/sessions)、[Claude Agent SDK 会话管理](https://code.claude.com/docs/en/agent-sdk/sessions)

这里值得关注的，仍然是开发者能控制什么。DSH 向开发者开放会话机制的组成和扩展方式：会话事件是模型上下文及多种产品视图的共同依据，可以围绕这些事件增加持久状态和视图，也能替换持久化实现。

比如，你通过插件增加了一项需要在恢复后保留的会话状态，就可以把它的变化写成事件，再从历史事件恢复相应状态。这样，插件体系与 Session Log 就连起来了：新增能力也能参与会话的记录与延续。

---

## 什么时候这些差异有价值

如果只是希望 Agent 帮你完成日常编码，实际效果仍然要结合模型、工具和任务来判断。能换 Agent Loop、能组合插件，并不能直接证明它写代码更强。

但当目标是构建自己的 Agent 产品时，这些设计就很具体了：运行方式不合适，可以调整机制；完整工具组合太多，可以选择精简组合；需要服务浏览器用户或自动化程序，可以组合不同入口；新增状态需要在下次会话恢复时保留，可以把它纳入会话事件。

Cordis 管理插件怎样协作和退出，插件组合决定应用和 Agent 具备哪些能力，Session Log 为上下文、状态和后续恢复提供依据。对我来说，这三部分共同说明了 DSH 值得关注的地方：

> 开发者不仅能决定 Agent 使用什么能力，也能决定这些能力怎样运行、怎样组合成产品，以及会话怎样继续。

DSH 当前仍处于开发者预览阶段。理解这套设计，首先是看清哪些部分可以由自己决定；至于这些选择能不能带来更好的任务表现，还需要在具体场景中验证。[项目状态](../README.md#developer-preview)

---

## 延伸阅读

- [DSH 架构说明](../docs/architecture.zh.md)：插件组合、运行过程和 Session Log 的完整关系。
- [Cordis 入门](../docs/cordis-primer.zh.md)：服务、事件、依赖和插件生命周期的基本机制。
- [SDK Minimal](../packages/bundle/sdk-minimal/README.md)：精简组合提供的工具及其使用条件。
- [Session 子系统](../docs/subsystems/session.zh.md)：会话事件、存储与恢复的进一步说明。

## 开发备注

无。