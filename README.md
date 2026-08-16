**DeepSeek Harness（dsh）**是深度求索（DeepSeek AI）开发的**开源 Agent Harness（智能体框架）**，采用 MIT 许可证，以 TypeScript 编写，于 2026 年 8 月正式开源，目前处于开发者预览阶段（官方提示未来将有破坏兼容性的变更）。

它不优化模型本身，而是优化模型运行的环境。核心哲学五个字：**一切皆插件（Everything is a Plugin）**。

模型、工具、技能、会话、沙箱、存储、循环（agent loop）、调度、UI 等**所有 Agent 能力均由插件提供**，通过 Cordis 内核的服务（Service）与事件（Event）彼此协作——开发者无需改动任何源码，就能在配置层选择、替换、扩展任一能力。

DeepSeek Harness 并非新的 DeepSeek 模型，也非一个单纯的 API 客户端。它是一套用来构建、运行和扩展智能体的 SDK 与应用框架，默认可以连接 DeepSeek 模型（也可方便地自定义连接其它模型），让模型读取项目、修改文件、运行命令、管理任务、分配子任务，并通过 Web UI、全屏终端、Headless 命令或自动化协议与用户交互。

放到 DeepSeek Harness 里的语境里，“一切皆插件”的意思就是：模型怎么接、工具怎么调用、会话如何保持、对话怎么记忆、交互怎么呈现，甚至 Agent 主循环本身，全都计划做成可以插拔的组件，把插件细颗粒度化这件事做到了极致，出发点是让 Agent 的运行足够透明，在 Trajectory 视图里，实现 Agent 的每一次运行都有迹可循，全部可回放、可分叉、可审计，从而大幅降低 Agent 体验和成本优化的复杂度。插件化颗粒度越细，黑盒越少，运行越透明，我们看的越清，Agent 的优化空间也就越大。


Cordis 不是 DeepSeek 临时起意造的，它有十年的社区血统：作者是 Shigma（史一凡），北大出身、DeepSeek 成员，也是知名聊天机器人框架 Koishi 的创始人。Koishi 2020 年发布，社区积累了 3000+ 插件，Cordis 就是 2022 年从 Koishi 里抽出来的插件微内核，Koishi 至今是它最大的实战验证案例。定位是"Meta-Framework for Modern JavaScript Applications"，一个通用的插件框架，处理依赖注入、作用域服务、生命周期清理这几件事，与 agent、与 LLM 都没有直接关系，在 DeepSeek Harness 出现之前就已经存在。Cordis 在开源社区有实际的使用者。最典型的是 Koishi，一个跨平台聊天机器人框架（QQ / Discord / Telegram / 微信都接）。聊天机器人场景天然是"几十个插件拼在一起、热插拔、随时改配置"，与 agent 的场景几乎相同，只是没有 LLM。DeepSeek Harness 把整个框架的源码 vendor 进自己的仓库，改了个 scope 叫 @deepseek-ai/cordis，然后把公司自己写的每一个包，都设成对它的 peer dependency。这比"使用一个第三方库"更进一步：整个产品都构建在 cordis 之上。

先对比几种常见的插件形态：传统 DI 容器、轻量钩子方案、Cordis。这三种方案各有取舍，值得摆在一起看。传统 DI 容器的三个缺口。 "主流"方案是一个 DI 容器加生命周期注解。你 bind 了一个 service，谁来负责 unbind 和 cleanup？传统 DI 要么不管（泄漏），要么要求你手写 lifecycle hook。LLM provider 热替换了，依赖它的 ToolRegistry 应该自动重启，传统 DI 做不到这种"依赖驱动的重载"。配置上，传统 DI 的配置在启动时读一次；而 cordis.yml 里一行就是一个插件实例，改配置直接触发 HMR。Pi也是Agent界的明星产品了，但Pi 的 Extension 走的是另一条路。 Pi 的自我定位是 "self extensible coding agent"，它的插件叫 Extension（扩展），基于 TS，在扩展点上通过钩子塞逻辑：没有依赖注入，没有插件间依赖管理（加载顺序即优先级），也不能卸载。Cordis 的插件是带依赖声明、生命周期、可逆副作用的"组件"：有依赖注入，有反应性 coeffects，插件间可以声明依赖，卸载时能完整逆转副作用。这最后一点——"可逆副作用"——是整篇文章的地基。Cordis 的设计论文把这件事上升到理论层面，核心问题问得很直接：有没有一种 programming model，能让动态本身具备类似进程那样的生命周期隔离？进程和容器的好处在于：kill 掉再启动，状态就清空了。代价是重启会丢掉进程内的 cache、connection、partial computation。插件系统希望不用重启，也能完成同样的清理。论文为此给了两个定义：时间可组合性：卸载组件时，组件对共享环境所做的修改必须被完整、安全地逆转——这要求追踪组件执行的每一次资源分配、事件注册和状态变更。空间可组合性：依赖变化时，相关组件自动激活或停用。时间可组合性是这篇文章的主线。剩下的事情——Fiber、effect、系统边界、preset、Code Mode——都在回答同一个问题：怎么在工程上实现"卸载 = 完整逆转"。Cordis 生态还带了几个配套包，随项目一起 vendor 进来：一个配置文件加载器（负责解析 cordis.yml 这类声明式配置），一个 include 插件（负责把一份配置当成子树挂载进来，preset 机制用的就是它改名后的版本），一个 HMR 模块（支持插件热替换），外加日志和定时器这两个基础设施插件。这几块拼起来，才能真正理解"everything is a plugin"。
