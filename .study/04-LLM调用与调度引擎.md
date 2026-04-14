# LLM 调用与调度引擎

## 一、LLM 调用层

**源码位置**: [LLM.java](../genie-backend/src/main/java/com/jd/genie/agent/llm/LLM.java)

LLM 类是整个系统与大语言模型交互的核心，封装了所有与 LLM 的通信逻辑。

### 1.1 初始化与配置

```java
public LLM(String modelName, String llmErp) {
    LLMSettings config = Config.getLLMConfig(modelName);
    this.model = config.getModel();
    this.maxTokens = config.getMaxTokens();
    this.temperature = config.getTemperature();
    this.apiKey = config.getApiKey();
    this.baseUrl = config.getBaseUrl();
    this.functionCallType = config.getFunctionCallType();
    this.maxInputTokens = config.getMaxInputTokens();
}
```

配置来源为 `application.yml` 中的 `llm.settings`，支持多模型配置：

```yaml
llm:
  default:
    base_url: 'https://api.openai.com'
    apikey: 'sk-xxx'
    model: gpt-4.1
    max_tokens: 16384
  settings: '{
    "claude-3-7-sonnet-v1": {
        "model": "claude-3-7-sonnet-v1",
        "max_tokens": 8192,
        "base_url": "...",
        "apikey": "..."
    }
  }'
```

### 1.2 两种调用方式

#### ask() - 纯文本对话

```java
public CompletableFuture<String> ask(
    AgentContext context,
    List<Message> messages,      // 对话消息
    List<Message> systemMsgs,    // 系统消息
    boolean stream,              // 是否流式
    Double temperature           // 温度
)
```

用于不需要工具调用的场景，如 SummaryAgent 的总结任务。

#### askTool() - 带工具调用的对话

```java
public CompletableFuture<ToolCallResponse> askTool(
    AgentContext context,
    List<Message> messages,
    Message systemMsgs,
    ToolCollection tools,        // 可用工具集合
    ToolChoice toolChoice,       // 工具选择策略
    Double temperature,
    boolean stream,
    int timeout
)
```

用于需要 LLM 决定调用工具的场景，如 PlanningAgent 和 ExecutorAgent。

### 1.3 两种函数调用格式

系统支持两种 LLM 函数调用格式：

| 格式 | 说明 | 适用模型 |
|------|------|---------|
| function_call | OpenAI 标准的 function calling | GPT 系列、DeepSeek 等 |
| struct_parse | 结构化文本解析 | 不支持 function calling 的模型 |

通过 `functionCallType` 配置项控制。

### 1.4 Claude 模型适配

系统内置了 Claude 模型的格式转换：

```java
// GPT 格式 → Claude 格式
public List<Map<String, Object>> gptToClaudeTool(List<Map<String, Object>> gptTools)
```

主要差异：
- Claude 使用 `input_schema` 而非 `parameters`
- Claude 的工具调用结果放在 `user` 角色消息中
- Claude 使用 `tool_use` / `tool_result` 类型

### 1.5 消息截断

当对话历史超过模型最大输入 token 数时，LLM 会自动截断消息：

```java
public List<Map<String, Object>> truncateMessage(
    AgentContext context, 
    List<Map<String, Object>> messages, 
    int maxInputTokens
)
```

截断策略：
1. 保留系统消息（第一条）
2. 从最新消息开始保留
3. 遇到非 user 消息时删除，确保从 user 消息开始
4. 直到 token 数不超过限制

### 1.6 流式输出

支持 SSE 流式输出，通过 OkHttp 的 SSE 实现：

- 非流式：一次性返回完整响应
- 流式：逐 token 输出，通过 Printer 发送给前端

## 二、调度引擎

### 2.1 入口控制器

**源码位置**: [GenieController.java](../genie-backend/src/main/java/com/jd/genie/controller/GenieController.java)

`/AutoAgent` 是系统的核心入口，处理流程如下：

```
1. 创建 SseEmitter（超时 1 小时）
2. 启动 SSE 心跳（10 秒间隔）
3. 注册 SSE 事件监听
4. 拼接输出样式提示词
5. 构建 AgentContext
6. 构建 ToolCollection
7. 根据请求类型获取对应的 Handler
8. 异步执行 Handler.handle()
9. 完成后关闭 SSE 连接
```

### 2.2 Agent 类型路由

**源码位置**: [AgentHandlerFactory.java](../genie-backend/src/main/java/com/jd/genie/service/impl/AgentHandlerFactory.java)

系统通过工厂模式根据 `agentType` 路由到不同的处理器：

| agentType | Handler | 模式 |
|-----------|---------|------|
| 3 (PLAN_SOLVE) | PlanSolveHandlerImpl | Plan & Solve |
| 5 (REACT) | ReactHandlerImpl | ReAct |

### 2.3 Plan & Solve 模式调度

**源码位置**: [PlanSolveHandlerImpl.java](../genie-backend/src/main/java/com/jd/genie/service/impl/PlanSolveHandlerImpl.java)

这是最核心的调度逻辑：

```
1. SOP 召回（可选）
2. 创建 PlanningAgent、ExecutorAgent、SummaryAgent
3. PlanningAgent.run() → 拆解任务，返回子任务列表
4. 循环：
   a. 解析子任务列表（<sep> 分隔）
   b. 如果只有一个子任务 → ExecutorAgent 直接执行
   c. 如果有多个子任务 → 并发创建多个 ExecutorAgent 执行
   d. 合并执行结果
   e. PlanningAgent.run(executorResult) → 更新计划或获取下一批任务
   f. 如果返回 "finish" → 调用 SummaryAgent 总结
5. 检查状态：IDLE（达到最大迭代）、ERROR（异常）
```

### 并发执行机制

当 PlanningAgent 返回多个可并行的子任务时：

```java
Map<String, String> tmpTaskResult = new ConcurrentHashMap<>();
CountDownLatch taskCount = ThreadUtil.getCountDownLatch(planningResults.size());
List<ExecutorAgent> slaveExecutors = new ArrayList<>();

for (String task : planningResults) {
    ExecutorAgent slaveExecutor = new ExecutorAgent(agentContext);
    slaveExecutor.getMemory().addMessages(executor.getMemory().getMessages());
    slaveExecutors.add(slaveExecutor);
    
    ThreadUtil.execute(() -> {
        String taskResult = slaveExecutor.run(task);
        tmpTaskResult.put(task, taskResult);
        taskCount.countDown();
    });
}
ThreadUtil.await(taskCount);

// 合并记忆
for (ExecutorAgent slaveExecutor : slaveExecutors) {
    for (int i = memoryIndex; i < slaveExecutor.getMemory().size(); i++) {
        executor.getMemory().addMessage(slaveExecutor.getMemory().get(i));
    }
}
```

### 2.4 ReAct 模式调度

**源码位置**: [ReactHandlerImpl.java](../genie-backend/src/main/java/com/jd/genie/service/impl/ReactHandlerImpl.java)

ReAct 模式更简洁：

```
1. 创建 ReactImplAgent、SummaryAgent
2. ReactImplAgent.run(query) → think-act 循环
3. SummaryAgent.summaryTaskResult() → 总结
4. 输出结果
```

### 2.5 SOP 召回

SOP（Standard Operating Procedure）召回是 Plan & Solve 模式的增强功能：

```java
private void handleSopRecall(AgentContext agentContext, AgentRequest request) {
    // 1. 调用 SOP 召回服务
    SopRecallResponse sopResponse = sopRecallService.sopRecall(requestId, query);
    // 2. 如果召回成功，注入 SOP 提示词
    String sopPrompt = agentContext.getSopPrompt().replace("{{sop}}", sopContent);
    agentContext.setSopPrompt(sopPrompt);
}
```

SOP 允许用户预定义分析流程，系统会基于预定义流程升级 Plan & Solve 模式为 SOPPlan 模式。

## 三、输出样式

系统支持多种输出样式，通过 `outputStyle` 参数控制：

| 样式 | 说明 | 追加提示词 |
|------|------|-----------|
| html | 网页版报告 | "以 html 展示" |
| docs | Markdown 文档 | "以 markdown 展示" |
| table | Excel 表格 | "以 excel 展示" |
| dataAgent | 数据分析 | 使用 ReportTool + DataAnalysisTool |

## 四、SSE 通信机制

### 后端 → 前端

通过 `SseEmitter` 实现服务端推送：

```java
SseEmitter emitter = new SseEmitter(60 * 60 * 1000L); // 1小时超时

// 心跳保活
ScheduledFuture<?> heartbeatFuture = startHeartbeat(emitter, requestId);

// 事件类型
printer.send("plan_thought", content);    // 规划思考
printer.send("plan", plan);               // 计划内容
printer.send("task", step);               // 任务步骤
printer.send("tool_thought", content);    // 工具思考
printer.send("tool_result", result);      // 工具结果
printer.send("task_summary", summary);    // 任务总结
printer.send("result", taskResult);       // 最终结果
```

### 前端 → 后端（MultiAgent 通道）

`MultiAgentServiceImpl` 提供了另一个 SSE 通道，用于前端通过 `/web/api/v1/gpt/queryAgentStreamIncr` 接口与后端通信：

1. 前端发送请求到 `/web/api/v1/gpt/queryAgentStreamIncr`
2. 后端构建 AgentRequest，调用本地 `/AutoAgent` 接口
3. 读取 `/AutoAgent` 的 SSE 响应
4. 通过 `AgentResponseHandler` 处理并转发给前端
