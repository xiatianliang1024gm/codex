# `codex-rs/core/src/codex.rs` 学习笔记

这份笔记的目标不是覆盖 `codex.rs` 的全部细节，而是帮助你快速建立主线理解，知道这个文件在 Codex 架构中的位置、核心数据结构、主循环，以及应该如何阅读。

源文件：

- [`codex-rs/core/src/codex.rs`](../codex-rs/core/src/codex.rs)

## 一句话定位

`codex.rs` 是 Codex 的会话内核和单轮执行主循环所在文件。

你可以把它理解成：

- `Codex`：对外暴露的 facade
- `Session`：真正的运行时容器
- `TurnContext`：某一轮执行时冻结下来的上下文快照
- `submission_loop`：处理外部提交的主循环
- `run_sampling_request` / `try_run_sampling_request`：处理模型流式响应的主循环

## 总体心智模型

```text
Codex::spawn
  -> Session::new
    -> 初始化 state/services/history/rollout/auth/mcp
    -> 启动 submission_loop

外部调用：
Codex::submit(Op)
  -> tx_sub.send(Submission)

后台线程：
submission_loop
  -> match Op
  -> handlers::user_input_or_turn(...)
    -> 准备 TurnContext
    -> run_sampling_request(...)
      -> built_tools(...)
      -> build_prompt(...)
      -> try_run_sampling_request(...)
        -> 读取模型流式事件 ResponseEvent
        -> 识别 assistant message / tool call / completed
        -> 执行工具
        -> 回写工具结果
      -> 如果需要 follow-up，继续下一轮
```

## 第一层：对外 API 是 `Codex`

关键位置：

- [`codex-rs/core/src/codex.rs#L332`](../codex-rs/core/src/codex.rs#L332)
- [`codex-rs/core/src/codex.rs#L378`](../codex-rs/core/src/codex.rs#L378)

`Codex` 是对外壳，不是业务逻辑中心。它主要提供：

- `spawn()`：创建并启动一个 session
- `submit()`：提交一个 `Op`
- `next_event()`：从事件队列取事件
- `shutdown_and_wait()`：关闭并等待后台循环退出

可以把它理解成：

```text
Codex = 给 CLI / TUI / app-server 调用的门面
```

## 第二层：真正核心状态是 `Session`

关键位置：

- [`codex-rs/core/src/codex.rs#L744`](../codex-rs/core/src/codex.rs#L744)

`Session` 是真正的运行时容器。最重要的字段有：

- `state: Mutex<SessionState>`
- `active_turn: Mutex<Option<ActiveTurn>>`
- `services: SessionServices`
- `tx_event`
- `agent_status`

理解方式：

- `SessionState`：会话级状态和配置
- `active_turn`：当前是否有正在执行的一轮
- `services`：各种运行时依赖，比如 model client、mcp、plugins、telemetry、state db
- `tx_event`：向外部发送事件
- `agent_status`：对外广播当前 agent 状态

一句话：

```text
Session = 运行中的 agent 容器
```

## 第三层：单轮执行的快照是 `TurnContext`

关键位置：

- [`codex-rs/core/src/codex.rs#L777`](../codex-rs/core/src/codex.rs#L777)
- [`codex-rs/core/src/codex.rs#L1252`](../codex-rs/core/src/codex.rs#L1252)

`TurnContext` 是这个文件里最值得学习的设计之一。它把一轮执行需要的内容都冻结下来，包括：

- `sub_id`
- `cwd`
- `model_info`
- `approval_policy`
- `sandbox_policy`
- `tools_config`
- `reasoning_effort`
- `reasoning_summary`
- `user_instructions`
- `developer_instructions`
- `dynamic_tools`

可以总结为：

```text
Session = 长生命周期状态
TurnContext = 单轮执行配置快照
```

这种分层很适合自己实现简化版 code agent。

## 第四层：`Session::new` 负责装配运行时

关键位置：

- [`codex-rs/core/src/codex.rs#L1344`](../codex-rs/core/src/codex.rs#L1344)

`Session::new` 很长，但第一遍不需要深究全部细节。它主要做的是：

1. 校验基础配置，比如 `cwd` 必须是绝对路径
2. 根据 `InitialHistory` 判断是新线程、恢复线程还是 fork
3. 初始化 rollout recorder / state db
4. 拉取 auth / MCP server / 历史元信息
5. 生成 startup warnings
6. 初始化 telemetry、model client、network proxy、tool 相关服务
7. 最终构造 `Session`

这里最值得学习的是装配方式：

- 并行初始化独立依赖
- 把运行时依赖收敛到 `SessionServices`
- 把会话级配置收敛到 `SessionConfiguration`

第一遍阅读时，先抓“初始化了哪些依赖”，不要陷进每个依赖的细节。

## 第五层：最外层事件分发循环是 `submission_loop`

关键位置：

- [`codex-rs/core/src/codex.rs#L4039`](../codex-rs/core/src/codex.rs#L4039)

这个函数是外层主循环。逻辑很直接：

```rust
while let Ok(sub) = rx_sub.recv().await {
    match sub.op {
        Op::Interrupt => ...
        Op::UserTurn => ...
        Op::ExecApproval => ...
        Op::Shutdown => ...
    }
}
```

这里要抓住一个关键认识：

```text
Codex core 是一个 submission dispatcher
```

也就是说，外部所有行为都会先变成 `Op`，再由这里分发到对应 handler。

如果将来你自己做简化版 agent，最小集通常只需要：

- `UserTurn`
- `Interrupt`
- `Shutdown`

## 第六层：`UserTurn` 是主线入口

关键位置：

- [`codex-rs/core/src/codex.rs#L4121`](../codex-rs/core/src/codex.rs#L4121)
- [`codex-rs/core/src/codex.rs#L4381`](../codex-rs/core/src/codex.rs#L4381)

在 `submission_loop` 中，`Op::UserInput` 和 `Op::UserTurn` 都会走到：

- `handlers::user_input_or_turn(...)`

虽然这个 handler 细节不在我们这次展开的重点片段里，但它承担的职责很明确：

1. 把用户输入转成 turn 级输入
2. 构造 `TurnContext`
3. 启动本轮任务
4. 最终进入 `run_sampling_request(...)`

所以真正执行 agent 一轮逻辑的函数，不在 `submit()`，而在后面的采样请求主线里。

### `handlers::user_input_or_turn(...)` 到底做了什么

关键位置：

- [`codex-rs/core/src/codex.rs#L4381`](../codex-rs/core/src/codex.rs#L4381)

这个函数是把外部的 `Op::UserTurn` 或 `Op::UserInput` 接到运行时主线上的桥。

它的逻辑可以压缩成：

```text
1. 从 Op 中提取 items 和 SessionSettingsUpdate
2. 调用 sess.new_turn_with_sub_id(...) 创建当前 turn 上下文
3. 尝试把输入注入正在运行的 turn
4. 如果当前没有 active turn，就真正 spawn 一个新 task
```

更具体一点：

1. 如果是 `Op::UserTurn`
它会把这些字段整理进 `SessionSettingsUpdate`：

- `cwd`
- `approval_policy`
- `sandbox_policy`
- `collaboration_mode`
- `reasoning_summary`
- `service_tier`
- `final_output_json_schema`
- `personality`

2. 然后调用：

- `sess.new_turn_with_sub_id(sub_id, updates).await`

这一步会基于当前 session 配置，生成这一轮的 `TurnContext`。

3. 随后先尝试：

- `sess.steer_input(items, None).await`

也就是说，如果当前已经有一轮正在跑，系统会优先尝试把新输入“导向”当前任务，而不是直接新起一个 task。

4. 如果返回：

- `SteerInputError::NoActiveTurn(items)`

才说明当前没有活跃 turn，于是系统会：

- `sess.refresh_mcp_servers_if_requested(...)`
- `sess.take_startup_regular_task().await.unwrap_or_default()`
- `sess.spawn_task(...)`

这条链的意义非常重要：

```text
UserTurn 不一定总是启动一个新任务
它会先尝试把输入注入当前运行中的 turn
只有没有 active turn 时，才真正 spawn 新 task
```

这也是 Codex 能支持“对当前执行中的 agent 进行 steering / 追加输入”的基础。

## 第七层：一轮执行的框架在 `run_sampling_request`

关键位置：

- [`codex-rs/core/src/codex.rs#L6141`](../codex-rs/core/src/codex.rs#L6141)
- [`codex-rs/core/src/codex.rs#L6117`](../codex-rs/core/src/codex.rs#L6117)
- [`codex-rs/core/src/codex.rs#L6279`](../codex-rs/core/src/codex.rs#L6279)

`run_sampling_request` 是“一轮 agent 工作”的框架代码。它的结构大致是：

1. `built_tools(...)`
构建这一轮实际可见的工具集合

2. `sess.get_base_instructions()`
拿到本轮 base instructions

3. `build_prompt(...)`
把输入、工具、instructions 拼成 `Prompt`

4. `ToolCallRuntime::new(...)`
创建工具调用运行时

5. 调用 `try_run_sampling_request(...)`
真正向模型发起流式请求并消费事件流

6. 如果失败，做 retry / transport fallback

所以可以这么记：

```text
run_sampling_request = 一轮请求的外框
try_run_sampling_request = 真正的模型事件处理循环
```

## 第八层：工具集合是动态构建的

关键位置：

- [`codex-rs/core/src/codex.rs#L6279`](../codex-rs/core/src/codex.rs#L6279)

`built_tools(...)` 做的事包括：

- 从 MCP connection manager 拿 MCP tools
- 合并 plugin / connector / app tools
- 根据当前输入和显式选择做过滤
- 最后交给 `ToolRouter::from_config(...)`

这说明工具集合不是静态写死的，而是按 session 和 turn 运行时装配出来的。

如果你要做简化版 code agent，可以把这层先缩减成：

```text
ToolRouter = read_file + shell + write_file
```

但设计思想值得保留：

```text
工具集合应该是可组合、可过滤、运行时生成的
```

## 第九层：真正的模型流处理在 `try_run_sampling_request`

关键位置：

- [`codex-rs/core/src/codex.rs#L6944`](../codex-rs/core/src/codex.rs#L6944)

这是整个文件最接近“agent loop 核心”的地方。

它先调用：

- `client_session.stream(...)`

拿到 `ResponseEvent` 的异步流，然后在循环中不断处理事件：

- `Created`
- `OutputItemAdded`
- `OutputItemDone`
- `OutputTextDelta`
- `Completed`
- `RateLimits`
- `ServerModel`
- `Reasoning*`

这里的关键认识是：

```text
模型不是一次性返回最终结果
而是连续吐流式事件
Codex 一边接收，一边更新状态，一边触发工具执行
```

## 工具调用真正触发在哪里

关键位置：

- [`codex-rs/core/src/codex.rs#L7022`](../codex-rs/core/src/codex.rs#L7022)
- [`codex-rs/core/src/codex.rs#L7058`](../codex-rs/core/src/codex.rs#L7058)
- [`codex-rs/core/src/stream_events_utils.rs#L158`](../codex-rs/core/src/stream_events_utils.rs#L158)

最关键的一跳发生在：

- `ResponseEvent::OutputItemDone(item)`

此时系统会调用：

- `handle_output_item_done(...)`

这个函数可能返回：

- `tool_future`

然后被塞进：

- `in_flight`

所以可以把这件事记成：

```text
Tool call trigger point = ResponseEvent::OutputItemDone
```

很多人第一次读会误以为工具调用是在 `OutputTextDelta` 里处理的，不是。

### `handle_output_item_done(...)` 是如何分流的

关键位置：

- [`codex-rs/core/src/stream_events_utils.rs#L158`](../codex-rs/core/src/stream_events_utils.rs#L158)
- [`codex-rs/core/src/stream_events_utils.rs#L201`](../codex-rs/core/src/stream_events_utils.rs#L201)
- [`codex-rs/core/src/stream_events_utils.rs#L254`](../codex-rs/core/src/stream_events_utils.rs#L254)

`handle_output_item_done(...)` 是整个主线里非常关键的一个“分流器”。

它处理一个已经完成的 `ResponseItem`，然后把它分成两大类：

1. 这是一个工具调用
2. 这不是工具调用，而是普通 assistant/reasoning/web search/image generation 项

它的结构可以压缩成：

```text
match ToolRouter::build_tool_call(item) {
  Ok(Some(call)) => 这是工具调用
  Ok(None) => 这是普通输出项
  Err(RespondToModel(...)) => 把错误包装成 tool output 回灌给模型
  Err(Fatal(...)) => 整轮失败
}
```

#### 分支 1：`Ok(Some(call))`

含义：

- 模型输出被识别成了一个 tool call

系统会做这些事：

1. 记录日志
2. 立即持久化这个 `item`
3. 调 `tool_runtime.handle_tool_call(...)`
4. 把返回的 future 放进 `tool_future`
5. 标记 `needs_follow_up = true`

这里最重要的一点是：

```text
模型发出的工具调用会先被持久化，再异步调度执行
```

这样即使 turn 后续被中断，历史和 rollout 仍然是完整的。

#### 分支 2：`Ok(None)`

含义：

- 这个输出项不是工具调用

系统会调用：

- `handle_non_tool_response_item(...)`

把它转成 `TurnItem`，然后：

- 如果之前没有 active item，就先发 started event
- 再发 completed event
- 记录 conversation items
- 提取 `last_agent_message`

所以这一支的作用是：

```text
把普通模型输出变成 UI 可消费的 turn item/event
```

#### 分支 3：`Err(FunctionCallError::RespondToModel(message))`

含义：

- 工具请求本身不合法，但这不是系统致命错误
- 应该把错误文本作为 tool output 重新喂回模型

系统会构造：

- `ResponseInputItem::FunctionCallOutput`

然后记录进 conversation history，并设置：

- `needs_follow_up = true`

这代表：

```text
某些工具错误不会直接让 turn 失败
而是被包装成 tool output 返还给模型，让模型自己决定下一步
```

这是一种很典型的 agent 设计。

#### 分支 4：`Err(FunctionCallError::Fatal(message))`

含义：

- 真正的致命错误

这时直接返回：

- `Err(CodexErr::Fatal(message))`

让上层终止当前 turn。

### `handle_non_tool_response_item(...)` 的作用

关键位置：

- [`codex-rs/core/src/stream_events_utils.rs#L254`](../codex-rs/core/src/stream_events_utils.rs#L254)

这个函数负责把普通 `ResponseItem` 转成 `TurnItem`。

它主要处理：

- `Message`
- `Reasoning`
- `WebSearchCall`
- `ImageGenerationCall`

这里你可以理解成：

```text
handle_output_item_done = 判定是否为 tool call
handle_non_tool_response_item = 把非工具项变成 UI 事件项
```

这两个函数一起构成了模型输出到运行时事件的核心桥梁。

## 为什么存在 `in_flight`

关键位置：

- [`codex-rs/core/src/codex.rs#L6976`](../codex-rs/core/src/codex.rs#L6976)

这里有：

```rust
let mut in_flight: FuturesOrdered<...>
```

说明工具调用不是简单的“发现一个 tool call 就立即阻塞执行到底”。成熟实现允许工具 future 被收集、并在后续 drain。

这和模型支持 parallel tool calls 的能力相关。

如果你以后做简化版，完全可以先做串行工具执行，但要知道这个实现已经把并行工具调用纳入设计。

## turn 在哪里结束

关键位置：

- [`codex-rs/core/src/codex.rs#L7148`](../codex-rs/core/src/codex.rs#L7148)

在 `ResponseEvent::Completed` 分支里，系统会：

- flush 流式文本段
- 更新 token usage
- 判断是否还需要 follow-up
- 返回 `SamplingRequestResult`

所以模型侧的一轮结束信号是：

```text
ResponseEvent::Completed
```

不是流关闭，也不是最后一段文本。

## `SamplingRequestResult` 的关键意义

关键位置：

- [`codex-rs/core/src/codex.rs#L6395`](../codex-rs/core/src/codex.rs#L6395)

这个结构里最重要的字段是：

- `needs_follow_up`

它的意义是：

- 如果模型只是完成回答，不需要 follow-up
- 如果模型调用工具之后产生了新的上下文，通常需要 follow-up

这正是 code agent 闭环的核心之一。

## 你可以暂时忽略的内容

初学 `codex.rs` 时，以下分支会严重干扰主线理解，可以先跳过：

- `plan_mode`
- `realtime_*`
- `review`
- `mcp`
- `memories`

先盯住普通主线：

```text
UserTurn -> TurnContext -> build_prompt -> stream ResponseEvent -> tool call -> completed
```

## 建议的阅读顺序

不要从头到尾硬读，按这个顺序更高效：

1. [`codex-rs/core/src/codex.rs#L332`](../codex-rs/core/src/codex.rs#L332)
读 `Codex` 结构和外部接口

2. [`codex-rs/core/src/codex.rs#L744`](../codex-rs/core/src/codex.rs#L744)
读 `Session`

3. [`codex-rs/core/src/codex.rs#L777`](../codex-rs/core/src/codex.rs#L777)
读 `TurnContext`

4. [`codex-rs/core/src/codex.rs#L1344`](../codex-rs/core/src/codex.rs#L1344)
读 `Session::new`

5. [`codex-rs/core/src/codex.rs#L4039`](../codex-rs/core/src/codex.rs#L4039)
读 `submission_loop`

6. [`codex-rs/core/src/codex.rs#L6117`](../codex-rs/core/src/codex.rs#L6117)
读 `build_prompt`

7. [`codex-rs/core/src/codex.rs#L6279`](../codex-rs/core/src/codex.rs#L6279)
读 `built_tools`

8. [`codex-rs/core/src/codex.rs#L6141`](../codex-rs/core/src/codex.rs#L6141)
读 `run_sampling_request`

9. [`codex-rs/core/src/codex.rs#L6944`](../codex-rs/core/src/codex.rs#L6944)
读 `try_run_sampling_request`

## 带着问题读会更快

建议每次阅读时只回答这些问题：

1. 状态在哪里？
答案：`Session` 和 `SessionState`

2. 单轮配置在哪里冻结？
答案：`TurnContext`

3. 外部请求从哪里进入？
答案：`submit(Op)` -> `submission_loop`

4. 模型流从哪里进入？
答案：`client_session.stream(...)`

5. 工具调用在哪个事件上触发？
答案：`ResponseEvent::OutputItemDone`

6. 一轮何时结束？
答案：`ResponseEvent::Completed`

7. 继续下一轮的条件是什么？
答案：`needs_follow_up`

8. `UserTurn` 一定会启动一个新任务吗？
答案：不一定，先尝试 `steer_input` 注入当前 active turn，没有 active turn 才 `spawn_task`

9. 工具错误一定会终止 turn 吗？
答案：不一定，很多工具错误会被包装成 `FunctionCallOutput` 回灌给模型

## 压缩版总结

`codex.rs` 的主线可以压缩成：

```text
spawn session
-> 接收 Submission
-> UserTurn 先尝试 steer_input，必要时再 spawn_task
-> 为一轮请求构造 TurnContext
-> 动态构造工具集合
-> 发起模型流式请求
-> 逐个处理 ResponseEvent
-> 在 OutputItemDone 处触发工具
-> 通过 handle_output_item_done 分流为工具调用或普通消息事件
-> 在 Completed 处结束本轮
-> 如果需要 follow-up，继续下一轮
```

## 下一步建议

如果要继续深入，优先补这两块：

1. `handlers::user_input_or_turn(...)`
看 `UserTurn` 如何真正进入 turn 执行

2. `handle_output_item_done(...)`
看模型输出如何被转换成工具执行和事件发射

当你把这两块也串上，`codex.rs` 的主线就基本完整了。

本笔记已经补上这两块，可以继续沿以下方向深入：

1. `sess.new_turn_with_sub_id(...)`
看 turn context 是怎么从 session state 里派生出来的

2. `sess.spawn_task(...)`
看一个 task 的生命周期和取消逻辑

3. `ToolRouter::build_tool_call(...)`
看模型输出是如何被识别成具体工具调用的

## 第十层：`new_turn_with_sub_id(...)` 如何生成一轮上下文

关键位置：

- [`codex-rs/core/src/codex.rs#L2209`](../codex-rs/core/src/codex.rs#L2209)
- [`codex-rs/core/src/codex.rs#L2270`](../codex-rs/core/src/codex.rs#L2270)
- [`codex-rs/core/src/codex.rs#L2486`](../codex-rs/core/src/codex.rs#L2486)

这组函数是 `UserTurn -> TurnContext` 之间最关键的桥。

### `new_turn_with_sub_id(...)`

它的流程可以压缩成：

```text
1. 用 SessionSettingsUpdate 更新 session_configuration
2. 如果配置非法，直接发 ErrorEvent
3. 如果 cwd 变化，刷新 shell snapshot
4. 调用 new_turn_from_configuration(...) 生成 TurnContext
```

这里要注意：

```text
Turn 开始前，session 级配置就已经被推进到新状态
```

也就是说，这不是“拿旧配置临时跑一轮”，而是先更新 session 的当前配置，再从新配置派生 turn。

### `new_turn_from_configuration(...)`

这一步更像“冻结一轮快照”，主要做：

1. `build_per_turn_config(...)`
2. 把 approval policy 同步给 MCP connection manager
3. 如果 sandbox policy 变化，通知 MCP server
4. 重新读取 `model_info`
5. 重新读取当前 cwd 对应的 `skills_outcome`
6. 调 `make_turn_context(...)`
7. 填充 realtime 状态、output schema、git enrichment task

最值得学的点是：

```text
TurnContext 不是简单 clone session 配置
而是在 turn 开始时重新解析 model、skills、sandbox、network 等运行时信息
```

### `new_default_turn_with_sub_id(...)`

这个版本不带额外更新，直接用当前 session configuration 生成 turn。

典型使用场景：

- `undo`
- `compact`
- `run_user_shell_command`

这类并非由普通 `UserTurn` 驱动的任务。

## 第十一层：`spawn_task(...)` 才是真正启动任务的地方

关键位置：

- [`codex-rs/core/src/tasks/mod.rs#L142`](../codex-rs/core/src/tasks/mod.rs#L142)
- [`codex-rs/core/src/tasks/mod.rs#L221`](../codex-rs/core/src/tasks/mod.rs#L221)
- [`codex-rs/core/src/tasks/mod.rs#L232`](../codex-rs/core/src/tasks/mod.rs#L232)

虽然调用点在 `codex.rs`，但 `spawn_task(...)` 的定义在：

- [`codex-rs/core/src/tasks/mod.rs`](../codex-rs/core/src/tasks/mod.rs)

这是把 turn context 和后台任务执行真正连起来的地方。

### `spawn_task(...)` 做了什么

它的流程可以压缩成：

```text
1. abort 当前所有旧任务，reason = Replaced
2. 清理 connector selection
3. 创建 cancellation token / done notify / telemetry timer / tracing span
4. 在 tokio task 中执行 SessionTask::run(...)
5. 任务结束后调用 on_task_finished(...)
6. 把 RunningTask 注册成当前 active_turn
```

这里最重要的两点是：

### 1. 新任务启动前一定会终止旧任务

开头就会执行：

- `abort_all_tasks(TurnAbortReason::Replaced)`

这说明同一个 session 同一时刻只允许一个 active turn。

### 2. 生命周期由 `spawn_task` 统一管理

具体任务实现只负责：

- `SessionTask::run(...)`

而 turn 的开始、结束、取消、telemetry、active_turn 注册，都由 `spawn_task` 统一处理。

这是一种很好的分层方式。

## 第十二层：`SessionTask` 是任务抽象

关键位置：

- [`codex-rs/core/src/tasks/mod.rs#L107`](../codex-rs/core/src/tasks/mod.rs#L107)

`SessionTask` trait 定义了每种任务的统一接口：

- `kind()`
- `span_name()`
- `run(...)`
- `abort(...)`

这意味着以下几类流程都被统一成任务：

- Regular chat
- Review
- Undo
- Compact
- Ghost snapshot
- User shell

所以：

```text
spawn_task 不关心任务内容
它只关心任务生命周期
```

## 第十三层：普通用户对话任务是 `RegularTask`

关键位置：

- [`codex-rs/core/src/tasks/regular.rs`](../codex-rs/core/src/tasks/regular.rs)

`RegularTask` 是普通 `UserTurn` 最终走到的任务实现。

它的 `run(...)` 很直接：

1. 拿到 `Session`
2. 重置 `server_reasoning_included`
3. 取可能存在的 prewarmed client session
4. 调用：
   - `run_turn(...)`

所以普通主线进一步收敛成：

```text
UserTurn
-> user_input_or_turn(...)
-> new_turn_with_sub_id(...)
-> spawn_task(...)
-> RegularTask::run(...)
-> run_turn(...)
-> run_sampling_request(...)
-> try_run_sampling_request(...)
```

这条链已经足够作为普通 agent 回合的主骨架。

## 第十四层：任务结束和中断是怎么处理的

关键位置：

- [`codex-rs/core/src/tasks/mod.rs#L232`](../codex-rs/core/src/tasks/mod.rs#L232)
- [`codex-rs/core/src/tasks/mod.rs#L406`](../codex-rs/core/src/tasks/mod.rs#L406)
- [`codex-rs/core/src/codex.rs#L3896`](../codex-rs/core/src/codex.rs#L3896)

### `on_task_finished(...)`

任务正常结束后，会：

1. 取消 git enrichment task
2. 从 `active_turn` 中移除当前任务
3. 取出 pending input
4. 计算本轮 token/tool metrics
5. 发出：
   - `EventMsg::TurnComplete`

所以可以记成：

```text
正常完成 = on_task_finished -> TurnComplete
```

### `abort_all_tasks(...)` / `handle_task_abort(...)`

任务被中断时，会：

1. cancel cancellation token
2. 等一小段时间让任务自行退出
3. 必要时强制 abort handle
4. 调用 task 自己的 `abort(...)`
5. 如果是 `Interrupted`，清理 unified exec / js_repl，并写入 interrupt marker
6. 发出：
   - `EventMsg::TurnAborted`

中断链可以记成：

```text
interrupt_task()
-> abort_all_tasks(Interrupted)
-> handle_task_abort(...)
-> TurnAborted
```

这里也能看出一个重要设计：

```text
正常完成和中断终止是两条明确分开的生命周期路径
```

## 更新后的完整主线图

现在可以把普通 `UserTurn` 的主线补全成：

```text
Codex::submit(Op::UserTurn)
  -> submission_loop
    -> handlers::user_input_or_turn(...)
      -> sess.new_turn_with_sub_id(...)
        -> new_turn_from_configuration(...)
          -> make_turn_context(...)
      -> 先尝试 steer_input(...)
      -> 如果没有 active turn:
        -> sess.spawn_task(...)
          -> RegularTask::run(...)
            -> run_turn(...)
              -> run_sampling_request(...)
                -> built_tools(...)
                -> build_prompt(...)
                -> try_run_sampling_request(...)
                  -> 消费 ResponseEvent 流
                  -> OutputItemDone -> handle_output_item_done(...)
                  -> 触发工具或普通消息事件
                  -> Completed
          -> on_task_finished(...)
            -> TurnComplete
```

## 再补两个关键认识

### 1. turn 和 task 不是完全同义

在代码里它们很接近，但职责不同：

- `TurnContext`：这一轮的上下文快照
- `SessionTask`：执行这一轮工作的任务实现

可以简单记成：

```text
Turn 更偏“上下文”
Task 更偏“执行器”
```

### 2. `steer_input(...)` 是非常关键的交互能力

关键位置：

- [`codex-rs/core/src/codex.rs#L3749`](../codex-rs/core/src/codex.rs#L3749)

它说明系统不是每次收到用户输入都无脑开新 turn，而是允许把输入注入当前活跃 turn 的 pending input 中。

这使系统可以支持：

- 中途追加说明
- 调整 agent 执行方向
- 继续已有任务而不是完全重开

## 更新后的阅读顺序

现在建议阅读顺序变成：

1. [`codex-rs/core/src/codex.rs#L332`](../codex-rs/core/src/codex.rs#L332)
2. [`codex-rs/core/src/codex.rs#L744`](../codex-rs/core/src/codex.rs#L744)
3. [`codex-rs/core/src/codex.rs#L777`](../codex-rs/core/src/codex.rs#L777)
4. [`codex-rs/core/src/codex.rs#L2209`](../codex-rs/core/src/codex.rs#L2209)
5. [`codex-rs/core/src/codex.rs#L4039`](../codex-rs/core/src/codex.rs#L4039)
6. [`codex-rs/core/src/codex.rs#L4381`](../codex-rs/core/src/codex.rs#L4381)
7. [`codex-rs/core/src/tasks/mod.rs#L142`](../codex-rs/core/src/tasks/mod.rs#L142)
8. [`codex-rs/core/src/tasks/regular.rs`](../codex-rs/core/src/tasks/regular.rs)
9. [`codex-rs/core/src/codex.rs#L6141`](../codex-rs/core/src/codex.rs#L6141)
10. [`codex-rs/core/src/codex.rs#L6944`](../codex-rs/core/src/codex.rs#L6944)
11. [`codex-rs/core/src/stream_events_utils.rs#L158`](../codex-rs/core/src/stream_events_utils.rs#L158)

## 接下来最值得继续补的地方

如果继续沿主线深入，优先级建议是：

1. `run_turn(...)`
看真正的一轮执行如何围绕 `run_sampling_request` 展开

2. `ToolRouter::build_tool_call(...)`
看模型输出是如何识别成具体工具调用的

3. `state::ActiveTurn` / `RunningTask` / `TurnState`
看 active turn、pending input、approval、permission grant 是如何存储的
