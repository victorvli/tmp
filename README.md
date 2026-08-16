时间可组合性是这篇文章的主线。剩下的事情——Fiber、effect、系统边界、preset、Code Mode——都在回答同一个问题：怎么在工程上实现"卸载 = 完整逆转"。Cordis 生态还带了几个配套包，随项目一起 vendor 进来：一个配置文件加载器（负责解析 cordis.yml 这类声明式配置），一个 include 插件（负责把一份配置当成子树挂载进来，preset 机制用的就是它改名后的版本），一个 HMR 模块（支持插件热替换），外加日志和定时器这两个基础设施插件。这几块拼起来，才能真正理解"everything is a plugin"。

Cordis 里，插件的类型定义是一个联合类型：type Plugin<T> = Plugin.Function<T> | Plugin.Constructor<T> | Plugin.Object<T>
裸函数、类、带 apply 方法的对象，三种写法最终都被解析成一个统一的回调。挂载一个插件调用 ctx.plugin(plugin, config)，内部先找有没有已经存在的 Runtime 记录（按回调函数身份做 key，同一个插件多次挂载共享同一条 Runtime），没有就新建，再造一个 Fiber，塞进这条 Runtime 的 fibers 列表里。Fiber 是插件实例的生命周期状态机，六个状态：PENDING、LOADING、ACTIVE、FAILED、DISPOSED、UNLOADING。插件声明了 inject: ['tools', 'shell']，对应的 Fiber 会停在 PENDING，直到这两个服务都在依赖链上出现，才真正跑这个插件的代码，进入 ACTIVE。这套等待机制是 Cordis 自己实现的依赖解析，插件作者不需要手写"等对方准备好"的轮询逻辑。这对应空间可组合性：依赖出现后，组件自动激活。卸载一个插件是否安全，由 effect 机制决定。插件注册的任何东西——事件监听、服务、定时器——都通过 ctx.effect() 登记。登记时立刻执行一次，返回的撤销函数被推进一个 disposables 列表。论文 5.1.1 指出：Cordis 中每一次上下文变更都通过唯一原语 ctx.effect 完成——提供服务、实例化组件、所有修改上下文的操作都归约成一次 ctx.effect 调用。用法长这样：ctx.effect(() => {
  // 做副作用：注册监听、开定时器、提供服务……
  return () => { /* 逆：撤销上面的操作 */ }   // 返回 disposer
})
callback 可以返回一个 disposer 函数、一个 Promise，或者一个（异步）迭代器，逐段 yield disposer——每做一步副作用就交一个撤销函数。


下面的时序图清晰地展示了从加载到卸载的完整流程：

sequenceDiagram
    participant Config as 配置/用户
    participant Cordis as Cordis 框架
    participant Fiber as Fiber (状态机)
    participant Plugin as 插件实例
    participant Effect as Effect (副作用)

    Config->>Cordis: 1. 声明加载插件
    Cordis->>Fiber: 2. 创建Fiber，状态: PENDING
    Note over Fiber: 依赖检查 (inject)
    Fiber->>Fiber: 3. 依赖就绪，状态: LOADING
    Fiber->>Plugin: 4. 执行 apply(ctx)
    Plugin->>Effect: 5. 注册资源 (ctx.effect/ctx.on)
    Effect-->>Plugin: 返回 disposer (撤销函数)
    Plugin-->>Fiber: 6. 加载完成，状态: ACTIVE
    Note over Plugin: 插件运行中

    Config->>Cordis: 7. 触发卸载 (HMR/移除/依赖丢失)
    Cordis->>Fiber: 8. 状态: UNLOADING
    Fiber->>Effect: 9. 逆序执行 disposer
    Effect-->>Fiber: 10. 资源清理完成
    Fiber->>Fiber: 11. 状态: DISPOSED

1. 核心：Fiber 状态机与依赖驱动
每个插件实例在运行时都被封装在一个叫做 Fiber 的执行单元中。它如同一个“生命周期管家”，精确控制着插件的状态流转。

六大核心状态：

PENDING（等待）：插件已声明，但其 inject 字段声明的依赖服务尚未就绪。这是防止加载顺序问题的关键。

LOADING（加载中）：所有依赖已满足，正在执行插件的 apply 函数。

ACTIVE（激活）：apply 函数执行完毕，插件正常运行。

FAILED（失败）：apply 函数执行或配置校验过程中抛出错误。

UNLOADING（卸载中）：正在执行清理工作。

DISPOSED（已销毁）：所有资源已释放，插件生命周期终结。

2. 关键机制：可逆副作用 (Effect)
DeepSeek Harness 能实现“干净”卸载的核心奥秘在于 “可逆副作用”。它要求插件对系统产生的任何影响（即副作用）都必须登记在案，并附带一个精确的“逆操作”（撤销函数）。

规范做法：插件不应该直接操作外部资源（如启动定时器、监听事件、注册工具）。所有操作都必须通过 Cordis 提供的 ctx.effect() API 或封装好的注册 API（如 ctx.on, ctx.tools.register）来完成。

撤销函数的注册：

typescript
// 官方文档示例 [citation:2]
export function apply(ctx: Context) {
  // 注册一个副作用，并返回其撤销函数
  ctx.effect(() => {
    // 1. 执行副作用：启动一个定时器
    const timer = setInterval(() => console.log('tick'), 200);
    
    // 2. 返回撤销函数：用于清理这个副作用
    return () => {
      clearInterval(timer); // 清理定时器
      console.log('heartbeat cleaned up');
    };
  });
}
这种模式将“创建”与“销毁”的逻辑放在一起，被称为“关注点局部性”。

3. 卸载与清理流程
当一个插件被卸载（无论是因为用户禁用、配置变更、热替换还是其依赖服务消失），Cordis 会执行一套严格的清理流程。

触发：Fiber 状态变为 UNLOADING。

逆序清理：Cordis 会找出该 Fiber 及其所有子 Fiber 在生命周期中注册的全部撤销函数，并按照注册的相反顺序依次执行。这被称为“后进先出”（LIFO）策略，确保了清理的顺序性。

级联卸载：如果一个插件通过 ctx.plugin() 挂载了子插件，那么父插件在卸载时会自动触发其所有子插件的卸载，形成一个递归的、完整的清理链条。

终结：所有撤销函数执行完毕后，Fiber 进入 DISPOSED 状态，插件实例可以被垃圾回收。

4. 生命周期中的特殊场景
热模块替换 (HMR)：这是“可逆副作用”带来的关键能力。当代码或配置变更时，Cordis 会先完整卸载（DISPOSED）旧插件，清理其所有副作用，然后再加载（PENDING -> ACTIVE）新插件。这个过程无需重启整个进程。

依赖驱动：如果一个 ACTIVE 状态的插件所依赖的服务被卸载，它也会自动进入 UNLOADING 并最终变为 DISPOSED 状态。待该服务重新出现后，它又可以自动重新加载。这保证了系统状态的一致性。
