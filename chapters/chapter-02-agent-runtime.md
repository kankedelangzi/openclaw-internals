# 第二章：Agent Runtime与Hook系统深度剖析

## 2.1 Agent Runtime整体架构

Agent Runtime是OpenClaw的核心执行引擎，负责管理AI Agent的完整生命周期。它不是一个单一的进程，而是一组协同工作的组件，包括消息处理管道、上下文管理器、工具调用引擎、记忆系统和Hook扩展点。

理解Agent Runtime的关键是从消息流的角度看待一切。当一条用户消息到达OpenClaw Gateway时，它经历了一个精心设计的处理管道：消息接收→消息解析→上下文构建→Agent推理→工具调用→响应生成→响应发送。每个阶段都可以被Hook拦截和修改，这正是OpenClaw高度可扩展的根源。

从代码结构看，Agent Runtime的核心在`packages/runtime/`目录下，其主要模块：

```
runtime/
├── src/
│   ├── index.ts              # Runtime主入口，导出核心类
│   ├── Agent.ts              # Agent类，管理单Agent状态
│   ├── MessagePipeline.ts    # 消息处理管道
│   ├── ContextManager.ts     # 上下文管理器
│   ├── ToolEngine.ts         # 工具调用引擎
│   ├── HookSystem.ts         # Hook注册与执行
│   └── MemoryBridge.ts        # 记忆系统桥接
```

### 2.1.1 Runtime初始化流程

Agent Runtime的初始化是理解整个系统的起点。初始化过程不仅创建对象，更建立了各组件之间的依赖关系和通信通道。

```typescript
// runtime/src/index.ts 核心初始化逻辑
export class AgentRuntime {
  private pipeline: MessagePipeline;
  private contextManager: ContextManager;
  private toolEngine: ToolEngine;
  private hookSystem: HookSystem;
  private memoryBridge: MemoryBridge;
  
  constructor(config: RuntimeConfig) {
    // 1. 初始化Hook系统（最早初始化，因为其他组件可能依赖Hook）
    this.hookSystem = new HookSystem();
    this.registerBuiltinHooks();
    
    // 2. 初始化上下文管理器
    this.contextManager = new ContextManager({
      maxTokens: config.maxContextTokens,
      systemPrompt: config.systemPrompt,
      hookSystem: this.hookSystem
    });
    
    // 3. 初始化工具引擎
    this.toolEngine = new ToolEngine({
      tools: config.tools || [],
      timeout: config.toolTimeout,
      retryPolicy: config.retryPolicy
    });
    
    // 4. 初始化消息管道（依赖其他组件）
    this.pipeline = new MessagePipeline({
      contextManager: this.contextManager,
      toolEngine: this.toolEngine,
      hookSystem: this.hookSystem,
      modelProvider: config.modelProvider
    });
    
    // 5. 初始化记忆桥接
    this.memoryBridge = new MemoryBridge({
      memorySystem: config.memorySystem,
      contextManager: this.contextManager
    });
    
    // 6. 注册配置变更监听
    config.onConfigChange((newConfig) => {
      this.handleConfigUpdate(newConfig);
    });
  }
  
  private registerBuiltinHooks(): void {
    // 注册内置Hook：日志、监控、错误处理
    this.hookSystem.register('pre:message', builtInHooks.logMessage);
    this.hookSystem.register('post:tool', builtInHooks.measureToolLatency);
    this.hookSystem.register('error', builtInHooks.errorReporter);
  }
}
```

这个初始化顺序是有意义的：Hook系统最先初始化，因为它为其他组件提供扩展能力；上下文管理器在工具引擎之前初始化，因为工具调用需要依赖上下文；消息管道最后初始化，它组合了所有其他组件。

### 2.1.2 消息处理管道

MessagePipeline是Runtime的核心，它定义了消息从进入到输出的完整流程。Pipeline采用职责链模式（Chain of Responsibility），每个处理器负责一个特定的任务。

```typescript
// runtime/src/MessagePipeline.ts
interface PipelineContext {
  message: IncomingMessage;
  agent: Agent;
  session: Session;
  history: Message[];
  tools: Tool[];
  response?: ModelResponse;
  error?: Error;
}

type PipelineHandler = (
  ctx: PipelineContext,
  next: () => Promise<void>
) => Promise<void>;

export class MessagePipeline {
  private handlers: PipelineHandler[] = [];
  
  constructor(private deps: PipelineDeps) {
    this.setupHandlers();
  }
  
  private setupHandlers(): void {
    // 管道处理顺序：按注册顺序执行
    // 每个handler调用next()将控制权传递给下一个handler
    
    // Handler 1: 消息接收与解析
    this.handlers.push(async (ctx, next) => {
      // 验证消息格式
      ctx.message = this.parseMessage(ctx.message.raw);
      
      // 触发pre:message Hook
      const preResult = await this.deps.hookSystem.emit(
        'pre:message',
        ctx.message,
        ctx
      );
      
      if (!preResult.continue) {
        ctx.response = preResult.response;
        return; // 短路，不继续处理
      }
      
      await next();
    });
    
    // Handler 2: 上下文构建
    this.handlers.push(async (ctx, next) => {
      // 构建当前消息的上下文
      const currentContext = await this.deps.contextManager.buildContext(
        ctx.session.id,
        ctx.message
      );
      
      // 合并历史消息和系统提示
      ctx.history = await this.mergeHistory(
        currentContext,
        ctx.session.config.maxHistory
      );
      
      // 触发pre:推理 Hook（上下文增强点）
      await this.deps.hookSystem.emit('pre:推理', ctx, {
        history: ctx.history,
        tools: ctx.tools
      });
      
      await next();
    });
    
    // Handler 3: Agent推理
    this.handlers.push(async (ctx, next) => {
      const modelResponse = await this.deps.modelProvider.complete({
        messages: ctx.history,
        tools: ctx.tools,
        model: ctx.agent.config.model,
        temperature: ctx.agent.config.temperature
      });
      
      ctx.response = modelResponse;
      await next();
    });
    
    // Handler 4: 推理结果处理（Tool Call检测）
    this.handlers.push(async (ctx, next) => {
      if (ctx.response.toolCalls && ctx.response.toolCalls.length > 0) {
        // 有工具调用，进入工具执行阶段
        await this.executeToolCalls(ctx);
      }
      
      await next();
    });
    
    // Handler 5: 响应生成后处理
    this.handlers.push(async (ctx, next) => {
      // 触发post:推理 Hook
      await this.deps.hookSystem.emit('post:推理', ctx.response, ctx);
      
      await next();
    });
    
    // Handler 6: 响应发送前处理
    this.handlers.push(async (ctx, next) => {
      // 触发post:响应 Hook（内容审核、格式转换）
      const postResult = await this.deps.hookSystem.emit(
        'post:响应',
        ctx.response,
        ctx
      );
      
      if (!postResult.continue) {
        ctx.response = postResult.response;
      }
      
      await next();
    });
    
    // Handler 7: 最终响应发送
    this.handlers.push(async (ctx, next) => {
      await this.sendResponse(ctx);
      await next();
    });
  }
  
  async process(rawMessage: unknown): Promise<ModelResponse> {
    const ctx: PipelineContext = {
      message: rawMessage as IncomingMessage,
      agent: this.resolveAgent(),
      session: await this.getOrCreateSession(),
      history: [],
      tools: this.deps.toolEngine.getTools()
    };
    
    // 串联执行所有handler
    await this.runPipeline(ctx, 0);
    
    return ctx.response!;
  }
  
  private async runPipeline(ctx: PipelineContext, index: number): Promise<void> {
    if (index >= this.handlers.length) {
      return;
    }
    
    const handler = this.handlers[index];
    await handler(ctx, () => this.runPipeline(ctx, index + 1));
  }
}
```

Pipeline的设计关键在于`next()`函数的调用模式。每个handler完成自己的处理后，调用`next()`将控制权传递给下一个handler。这种模式允许在处理流程的任意位置插入逻辑：可以在`next()`之前做预处理（pre-hook效果），也可以在`next()`之后做后处理（post-hook效果）。

当某个handler决定短路时（比如权限检查失败），它直接返回而不调用`next()`，管道就此终止。这种设计使得横切关注点（cross-cutting concerns）如日志、监控、权限检查等可以优雅地集成到主处理流程中。

## 2.2 Hook系统深度解析

### 2.2.1 Hook类型体系

OpenClaw的Hook系统是其可扩展性的核心。与其让开发者修改核心代码来定制行为，不如提供扩展点（Extension Points），让开发者通过Hook在特定时机插入自己的逻辑。

Hook类型按触发时机分类：

`pre:message` Hook在消息被处理之前触发，常用于权限检查、内容过滤、消息验证。如果Hook返回`continue: false`，消息不会被进一步处理。

`pre:推理` Hook在推理开始前触发，此时上下文已构建完毕，Hook可以修改上下文内容、注入变量、做意图分析。

`post:推理` Hook在推理完成后触发，可以检查推理结果、记录日志、决定是否需要重试。

`pre:工具` Hook在工具调用前触发，可以验证参数、记录调用意图、检查速率限制。

`post:工具` Hook在工具调用后触发，是最常用的Hook类型——self-improving-agent用它记录命令执行失败，监控用它记录工具延迟。

`error` Hook在任何阶段发生异常时触发，用于统一错误处理和诊断。

### 2.2.2 Hook注册与执行机制

Hook系统的核心是注册表和事件分发机制：

```typescript
// runtime/src/HookSystem.ts
type HookHandler = (
  payload: unknown,
  context: HookContext
) => Promise<HookResult>;

interface HookResult {
  continue: boolean;
  response?: ModelResponse;
  modifiedPayload?: unknown;
  metadata?: Record<string, unknown>;
}

interface HookRegistration {
  name: string;
  handler: HookHandler;
  priority: number;  // 数字越小越先执行
  options: HookOptions;
}

export class HookSystem {
  private handlers: Map<string, HookRegistration[]> = new Map();
  private globalHandlers: HookRegistration[] = [];
  
  // 注册Hook
  register(
    eventType: string,
    handler: HookHandler,
    options: Partial<HookOptions> = {}
  ): void {
    const registration: HookRegistration = {
      name: options.name || handler.name || 'anonymous',
      handler,
      priority: options.priority ?? 100,
      options: { ...defaultHookOptions, ...options }
    };
    
    if (options.global) {
      this.globalHandlers.push(registration);
      this.sortHandlers(this.globalHandlers);
    } else {
      if (!this.handlers.has(eventType)) {
        this.handlers.set(eventType, []);
      }
      this.handlers.get(eventType)!.push(registration);
      this.sortHandlers(this.handlers.get(eventType)!);
    }
  }
  
  // 触发Hook链
  async emit(
    eventType: string,
    payload: unknown,
    context: HookContext
  ): Promise<HookResult> {
    const registrations = [
      ...this.globalHandlers,
      ...(this.handlers.get(eventType) || [])
    ];
    
    let currentPayload = payload;
    let result: HookResult = { continue: true };
    
    for (const reg of registrations) {
      // 跳过已禁用的Hook
      if (!reg.options.enabled) {
        continue;
      }
      
      // 检查条件
      if (reg.options.condition && !await reg.options.condition(context)) {
        continue;
      }
      
      // 频率限制
      if (reg.options.throttleMs) {
        const lastRun = this.lastRunTimes.get(reg.name);
        if (lastRun && Date.now() - lastRun < reg.options.throttleMs) {
          continue;
        }
        this.lastRunTimes.set(reg.name, Date.now());
      }
      
      try {
        const hookResult = await this.executeHook(reg, currentPayload, context);
        
        if (hookResult.modifiedPayload) {
          currentPayload = hookResult.modifiedPayload;
        }
        
        if (!hookResult.continue) {
          return hookResult;
        }
        
        result = hookResult;
      } catch (error) {
        // Hook执行出错
        console.error(`Hook ${reg.name} failed:`, error);
        
        if (reg.options.required) {
          throw error;
        }
      }
    }
    
    return result;
  }
  
  private async executeHook(
    reg: HookRegistration,
    payload: unknown,
    context: HookContext
  ): Promise<HookResult> {
    const startTime = Date.now();
    
    try {
      const result = await reg.handler(payload, context);
      const duration = Date.now() - startTime;
      
      // 记录Hook执行指标
      this.recordMetric(reg.name, eventType, duration, result.continue);
      
      return result;
    } catch (error) {
      this.recordMetric(reg.name, eventType, Date.now() - startTime, false, error);
      throw error;
    }
  }
  
  private sortHandlers(handlers: HookRegistration[]): void {
    handlers.sort((a, b) => a.priority - b.priority);
  }
}
```

Hook执行顺序由priority决定，数字越小越先执行。内置Hook通常priority=10，用户Hook默认priority=100，允许内置Hook在任何用户Hook之前执行。

throttleMs选项实现频率限制，防止Hook被过度触发。self-improving-agent的activator.sh使用60分钟的throttle，防止在学习时机未成熟时频繁提醒。

### 2.2.3 Hook上下文传递

HookContext是Hook系统的数据传递载体，它包含了Hook执行时所需的所有信息：

```typescript
// runtime/src/types.ts
interface HookContext {
  // 会话信息
  session: {
    id: string;
    agentId: string;
    channel: string;
    peer: string;
  };
  
  // Agent配置
  agent: {
    id: string;
    name: string;
    workspace: string;
    model: ModelConfig;
  };
  
  // 配置（Gateway层级）
  config: GatewayConfig;
  
  // 当前时间
  timestamp: Date;
  
  // CancellationToken用于取消长时间操作
  signal: AbortSignal;
  
  // 工具注册表
  tools: ToolRegistry;
  
  // 记忆系统引用
  memory: MemorySystem;
}
```

Hook可以利用context访问Agent配置、记忆系统、工具注册表。这使得Hook不仅能做拦截，还可以主动查询状态、调用其他工具、甚至修改Agent行为。

## 2.3 上下文管理机制

### 2.3.1 上下文构建策略

上下文管理器（ContextManager）负责在每次推理调用前构建完整的上下文。上下文大小直接影响推理质量和成本，因此构建策略至关重要。

```typescript
// runtime/src/ContextManager.ts
export class ContextManager {
  private systemPrompts: SystemPrompt[];
  private memoryBridge: MemoryBridge;
  
  constructor(private config: ContextConfig) {}
  
  async buildContext(
    sessionId: string,
    incomingMessage: Message
  ): Promise<Context> {
    const blocks: ContextBlock[] = [];
    
    // 1. 系统级上下文（始终注入）
    blocks.push(...this.buildSystemContext());
    
    // 2. 用户级上下文（从记忆加载）
    blocks.push(...await this.loadUserContext(sessionId));
    
    // 3. 当前消息
    blocks.push({
      role: 'user',
      content: incomingMessage.text,
      timestamp: incomingMessage.timestamp
    });
    
    // 4. 工具描述（如果需要）
    if (this.shouldIncludeTools(sessionId)) {
      blocks.push(...this.buildToolsContext());
    }
    
    // 5. 截断到最大token数
    return this.truncateAndAssemble(blocks);
  }
  
  private buildSystemContext(): ContextBlock[] {
    return this.systemPrompts.map(prompt => ({
      role: 'system' as const,
      content: prompt.content,
      metadata: {
        source: 'system',
        priority: prompt.priority
      }
    }));
  }
  
  private async loadUserContext(sessionId: string): Promise<ContextBlock[]> {
    const session = await this.getSession(sessionId);
    const memories = await this.memoryBridge.search(
      session.userId,
      {
        type: 'relevant',
        query: session.currentIntent,
        limit: this.config.maxMemoryBlocks
      }
    );
    
    return memories.map(m => ({
      role: 'system' as const,
      content: `相关记忆: ${m.content}`,
      metadata: {
        source: 'memory',
        memoryId: m.id,
        relevance: m.score
      }
    }));
  }
}
```

系统提示按优先级分层：高优先级系统提示（如身份定义）始终在前面，低优先级（如特定场景提示）在后面。当上下文超长时，从后往前截断，保护核心系统提示不被丢弃。

### 2.3.2 上下文截断策略

上下文截断是成本控制的关键。粗暴的截断会丢失重要信息，精细的截断需要理解语义边界。

```typescript
// runtime/src/ContextManager.ts (截断策略)
private truncateAndAssemble(blocks: ContextBlock[]): Context {
  let totalTokens = this.countTokens(blocks);
  const maxTokens = this.config.maxContextTokens;
  
  if (totalTokens <= maxTokens) {
    return { blocks, totalTokens };
  }
  
  // 策略：保护系统提示，按相关性排序其他块
  const systemBlocks = blocks.filter(b => b.metadata?.source === 'system');
  const otherBlocks = blocks.filter(b => b.metadata?.source !== 'system');
  
  // 按相关性排序（relevance分数）
  otherBlocks.sort((a, b) => 
    (b.metadata?.relevance ?? 0) - (a.metadata?.relevance ?? 0)
  );
  
  // 从低相关性开始移除
  const systemTokens = this.countTokens(systemBlocks);
  const availableForOthers = maxTokens - systemTokens;
  
  const selectedBlocks: ContextBlock[] = [...systemBlocks];
  let currentTokens = systemTokens;
  
  for (const block of otherBlocks) {
    const blockTokens = this.countTokens([block]);
    
    if (currentTokens + blockTokens <= availableForOthers) {
      selectedBlocks.push(block);
      currentTokens += blockTokens;
    } else {
      // 尝试摘要后加入
      const summary = await this.summarizeBlock(block);
      const summaryTokens = this.countTokens([summary]);
      
      if (currentTokens + summaryTokens <= availableForOthers) {
        selectedBlocks.push(summary);
        currentTokens += summaryTokens;
      }
      // 如果摘要也超，直接跳过
    }
  }
  
  return { blocks: selectedBlocks, totalTokens: currentTokens };
}
```

这个策略的核心思想是：系统提示是神圣不可侵犯的，用户消息和记忆则在截断范围内按相关性取舍。当整块无法容纳时，尝试摘要后保留，而非直接丢弃。

## 2.4 Pi Model Discovery机制

### 2.4.1 模型路由策略

Pi Model Discovery是OpenClaw的智能模型路由机制，它根据任务特征选择最合适的模型。这是OpenClaw"品类管理思维"的技术实现——不同任务用不同的模型处理。

```typescript
// runtime/src/PiModelDiscovery.ts
interface TaskFeatures {
  isCodeTask: boolean;        // 代码生成/调试
  isLongContext: boolean;      // 超长上下文
  isCostSensitive: boolean;   // 成本敏感
  requiresVision: boolean;     // 需要视觉能力
  isRealtime: boolean;        // 实时性要求
  latencyRequirement?: number; // 最大延迟ms
}

interface ModelSelection {
  provider: string;
  model: string;
  reason: string;
}

export class PiModelDiscovery {
  private modelProfiles: Map<string, ModelProfile>;
  private fallbackChain: string[];
  
  constructor(config: DiscoveryConfig) {
    this.modelProfiles = new Map(Object.entries(config.profiles));
    this.fallbackChain = config.fallbackChain || ['primary', 'secondary', 'tertiary'];
  }
  
  async discover(task: Task): Promise<ModelSelection> {
    const features = this.extractFeatures(task);
    
    // 规则1：代码任务用专用模型
    if (features.isCodeTask) {
      return {
        provider: 'deepseek',
        model: 'deepseek-coder-v2',
        reason: 'Code task detected, using specialized coder model'
      };
    }
    
    // 规则2：视觉任务用多模态模型
    if (features.requiresVision) {
      return {
        provider: 'anthropic',
        model: 'claude-3-5-sonnet-vision',
        reason: 'Vision capability required'
      };
    }
    
    // 规则3：超长上下文用上下文窗口大的模型
    if (features.isLongContext) {
      return {
        provider: 'anthropic',
        model: 'claude-3-5-sonnet-200k',
        reason: 'Long context requires large context window'
      };
    }
    
    // 规则4：成本敏感用小模型
    if (features.isCostSensitive) {
      return {
        provider: 'ollama',
        model: 'llama3:70b',
        reason: 'Cost-sensitive task, using local model'
      };
    }
    
    // 规则5：实时性要求高用低延迟模型
    if (features.latencyRequirement && features.latencyRequirement < 1000) {
      return {
        provider: 'openai',
        model: 'gpt-4o-mini',
        reason: 'Low latency required for realtime interaction'
      };
    }
    
    // 默认：主模型
    return {
      provider: this.fallbackChain[0],
      model: this.getDefaultModel(),
      reason: 'Default model selection'
    };
  }
  
  private extractFeatures(task: Task): TaskFeatures {
    const description = task.description || '';
    const messages = task.messages || [];
    
    // 代码任务检测：描述中包含代码相关关键词
    const codeKeywords = /code|function|class|debug|implement|script|programming/i;
    const isCodeTask = codeKeywords.test(description);
    
    // 长上下文检测：消息数量或token数超过阈值
    const isLongContext = messages.length > 20 || 
      this.estimateTokens(messages) > 50000;
    
    // 成本敏感检测：显式标记或特定关键词
    const isCostSensitive = 
      task.options?.costSensitive === true ||
      /simple|quick|brief|short/i.test(description);
    
    // 视觉检测：消息中包含图片
    const requiresVision = task.messages?.some(
      m => m.content && typeof m.content === 'object' && 'image' in m.content
    );
    
    return {
      isCodeTask,
      isLongContext,
      isCostSensitive,
      requiresVision,
      isRealtime: task.options?.realtime ?? false,
      latencyRequirement: task.options?.maxLatency
    };
  }
}
```

这个模型路由逻辑体现了"让合适的模型做合适的事"原则：代码任务用DeepSeek Coder这样的专业模型、长上下文用Claude 200k、成本敏感任务用本地Ollama模型。

### 2.4.2 模型降级与重试

当首选模型不可用时，Pi Model Discovery支持降级链：

```typescript
// runtime/src/PiModelDiscovery.ts (降级机制)
export class PiModelDiscovery {
  async discoverWithFallback(task: Task): Promise<ModelSelection> {
    const selection = await this.discover(task);
    
    // 尝试首选模型
    try {
      await this.validateModelAvailability(selection);
      return selection;
    } catch (error) {
      console.warn(`Primary model ${selection.model} unavailable, trying fallback`);
    }
    
    // 尝试降级链
    for (const fallback of this.fallbackChain.slice(1)) {
      const fallbackSelection = {
        ...selection,
        provider: fallback,
        reason: `${selection.reason} (via fallback)`
      };
      
      try {
        await this.validateModelAvailability(fallbackSelection);
        return fallbackSelection;
      } catch (error) {
        console.warn(`Fallback model ${fallback} also unavailable`);
      }
    }
    
    // 所有模型都不可用，返回最后的选择（让它在实际调用时失败）
    return {
      ...selection,
      reason: 'No available model in fallback chain'
    };
  }
  
  private async validateModelAvailability(
    selection: ModelSelection
  ): Promise<void> {
    const provider = this.getProvider(selection.provider);
    await provider.healthCheck(selection.model);
  }
}
```

降级链的好处是：网络抖动或服务临时不可用时，系统自动切换到备用模型，用户几乎感知不到。只有当降级链全部失败时，才向用户报告错误。

## 2.5 工具调用引擎

### 2.5.1 工具执行生命周期

ToolEngine管理工具的注册、调用和结果处理。当模型输出包含tool_calls时，ToolEngine接管执行：

```typescript
// runtime/src/ToolEngine.ts
export class ToolEngine {
  private tools: Map<string, Tool> = new Map();
  private retryPolicy: RetryPolicy;
  
  constructor(config: ToolEngineConfig) {
    this.retryPolicy = new RetryPolicy(config.retryPolicy);
    this.registerTools(config.tools);
  }
  
  async executeToolCall(
    call: ToolCall,
    context: HookContext
  ): Promise<ToolResult> {
    const tool = this.tools.get(call.name);
    
    if (!tool) {
      throw new ToolNotFoundError(`Tool '${call.name}' not found`);
    }
    
    // 触发pre:工具 Hook
    const preResult = await context.hookSystem.emit(
      'pre:工具',
      { call, tool },
      context
    );
    
    if (!preResult.continue) {
      return {
        success: false,
        error: preResult.response as string,
        skipped: true
      };
    }
    
    // 执行工具（带重试）
    let lastError: Error;
    
    for (let attempt = 0; attempt <= this.retryPolicy.maxRetries; attempt++) {
      try {
        const startTime = Date.now();
        
        const result = await this.executeWithTimeout(
          tool,
          call.arguments,
          context.signal
        );
        
        const duration = Date.now() - startTime;
        
        // 触发post:工具 Hook
        await context.hookSystem.emit(
          'post:工具',
          {
            call,
            result,
            duration,
            attempt
          },
          context
        );
        
        return {
          success: true,
          result,
          duration,
          attempt
        };
      } catch (error) {
        lastError = error as Error;
        
        if (!this.retryPolicy.shouldRetry(error, attempt)) {
          break;
        }
        
        const delay = this.retryPolicy.getDelay(attempt);
        await this.sleep(delay);
      }
    }
    
    // 所有重试都失败
    return {
      success: false,
      error: lastError!.message,
      attempt: this.retryPolicy.maxRetries + 1
    };
  }
  
  private async executeWithTimeout(
    tool: Tool,
    args: Record<string, unknown>,
    signal: AbortSignal
  ): Promise<unknown> {
    const timeout = setTimeout(() => {
      signal.dispatchEvent(new AbortEvent('timeout'));
    }, tool.timeout || 30000);
    
    try {
      return await tool.execute(args, { signal });
    } finally {
      clearTimeout(timeout);
    }
  }
}
```

重试策略是ToolEngine的核心能力之一。网络请求可能因为临时故障失败，合理的重试能显著提高系统健壮性：

```typescript
// runtime/src/RetryPolicy.ts
export class RetryPolicy {
  constructor(private config: RetryPolicyConfig) {}
  
  shouldRetry(error: Error, attempt: number): boolean {
    if (attempt >= this.config.maxRetries) {
      return false;
    }
    
    // 网络错误重试
    if (error instanceof NetworkError) {
      return true;
    }
    
    // 429 Too Many Requests 重试（等待冷却）
    if (error instanceof RateLimitError) {
      return true;
    }
    
    // 超时不重试（没有意义）
    if (error instanceof TimeoutError) {
      return false;
    }
    
    // 其他服务器错误重试
    if (error instanceof ServerError && error.statusCode >= 500) {
      return true;
    }
    
    return false;
  }
  
  getDelay(attempt: number): number {
    // 指数退避：1s, 2s, 4s, 8s...
    const exponentialDelay = Math.min(
      this.config.initialDelayMs * Math.pow(2, attempt),
      this.config.maxDelayMs
    );
    
    // 添加随机抖动（防止惊群效应）
    const jitter = Math.random() * 0.3 * exponentialDelay;
    
    return exponentialDelay + jitter;
  }
}
```

指数退避配合随机抖动是业界标准做法：重试间隔随失败次数指数增长，但随机性防止多个客户端同时重试造成流量尖峰。

## 2.6 Runtime与其他系统的协作

### 2.6.1 与Gateway的协作

Agent Runtime运行在Gateway进程内，通过事件总线与Gateway其他组件通信：

```typescript
// Gateway通过以下方式与Runtime交互：
// 1. Runtime实例化时注入Gateway配置
const runtime = new AgentRuntime({
  config: gatewayConfig.agent,
  modelProvider: gatewayConfig.modelProvider,
  memorySystem: gatewayConfig.memorySystem,
  
  // Gateway的eventBus用于跨组件通信
  eventBus: gateway.eventBus,
  
  // 配置变更监听
  onConfigChange: (newConfig) => {
    runtime.handleConfigUpdate(newConfig);
  }
});

// 2. Runtime通过eventBus报告状态
runtime.on('tool-call', (event) => {
  gateway.eventBus.publish('metrics:tool-call', {
    tool: event.toolName,
    duration: event.duration,
    success: event.success
  });
});

// 3. Gateway负责生命周期管理
process.on('SIGTERM', () => {
  runtime.shutdown();
});
```

这种设计使得Runtime可以独立测试，但在运行时与Gateway紧密集成。eventBus解耦了组件间通信，Runtime不需要知道Gateway的具体实现，只要通过eventBus发布事件即可。

### 2.6.2 与记忆系统的协作

Runtime通过MemoryBridge与记忆系统交互，实现上下文的持久化和检索：

```typescript
// runtime/src/MemoryBridge.ts
export class MemoryBridge {
  constructor(private deps: MemoryBridgeDeps) {}
  
  async saveInteraction(
    sessionId: string,
    interaction: Interaction
  ): Promise<void> {
    // 序列化交互内容
    const record = {
      sessionId,
      userMessage: interaction.userMessage,
      assistantMessage: interaction.assistantMessage,
      toolCalls: interaction.toolCalls,
      timestamp: new Date().toISOString(),
      embedding: await this.deps.embeddingService.embed(
        interaction.userMessage.text
      )
    };
    
    // 写入向量数据库
    await this.deps.memorySystem.add(record);
  }
  
  async search(
    userId: string,
    options: SearchOptions
  ): Promise<Memory[]> {
    const queryEmbedding = await this.deps.embeddingService.embed(
      options.query
    );
    
    // 向量相似度搜索
    const candidates = await this.deps.vectorStore.search(
      queryEmbedding,
      { limit: options.limit }
    );
    
    // RRR重排（Retrieval-Rerank-Refine）
    const reranked = await this.rerankResults(
      candidates,
      options.query,
      options.sessionContext
    );
    
    return reranked;
  }
}
```

MemoryBridge是Runtime与Memory系统的适配层。它处理嵌入生成、向量搜索结果重排等细节，让Runtime可以用简单接口操作复杂的记忆系统。

理解Agent Runtime的工作原理，对于调试OpenClaw行为、设计自定义扩展、规划系统容量都至关重要。Hook系统提供了无侵入的扩展能力，Pi Model Discovery实现了资源的精准配置，上下文管理确保了每次推理的质量与成本的平衡。这些机制共同构成了OpenClaw作为AI Agent运行平台的核心竞争力。
