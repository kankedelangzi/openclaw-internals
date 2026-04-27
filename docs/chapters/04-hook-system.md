# 第四章：Hook系统与事件驱动架构

Hook系统是OpenClaw事件驱动能力的核心支柱。它允许开发者在Agent执行过程中的关键节点插入自定义逻辑，实现日志记录、错误捕获、自动化学习等能力。掌握Hook系统，就掌握了OpenClaw的"神经末梢"。

## 4.1 Hook全景：六大类型与触发时机

OpenClaw的Hook系统覆盖了Agent运行时的完整生命周期，按触发阶段分为六大类：

| Hook类型 | 触发时机 | 典型用途 |
|----------|----------|----------|
| **Start Hooks** | Gateway/Session启动时 | 初始化、日志配置 |
| **Stop Hooks** | Gateway/Session关闭时 | 资源清理、状态保存 |
| **Auth Hooks** | 认证流程中 | 权限校验、IP白名单 |
| **Message Hooks** | 消息接收/发送时 | 内容审核、格式转换 |
| **ToolUse Hooks** | 工具调用前后 | 错误捕获、日志记录 |
| **PromptSubmit Hooks** | Prompt发送给模型前 | 上下文注入、Prompt增强 |

### 4.1.1 Hook执行时机图谱

```
用户消息
    │
    ▼
┌─────────────────────────────────────┐
│         MESSAGE HOOKS               │
│  message.received ──────────────────►│
│    │                                │
│    ▼                                │
│  消息解析与验证                      │
│    │                                │
│    ▼                                │
│  MESSAGE HOOKS                      │
│  message.parse ─────────────────────►│
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│         AUTH HOOKS                  │
│  auth.user ─────────────────────────►│
│    │                                │
│    ▼                                │
│  权限校验                            │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│      PROMPT SUBMIT HOOKS            │
│  prompt.submit ─────────────────────►│
│    │                                │
│    ▼                                │
│  Prompt构建 + 上下文注入             │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│         AGENT EXECUTION             │
│                                     │
│  ┌──────────────────────────────┐   │
│  │      TOOL USE HOOKS          │   │
│  │  tool.use.pre ──────────────►│   │
│  │    │                          │   │
│  │    ▼                          │   │
│  │  工具执行                      │   │
│  │    │                          │   │
│  │  tool.use.post ◄──────────────│   │
│  └──────────────────────────────┘   │
│                                     │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│         STOP HOOKS                  │
│  agent.shutdown ────────────────────►│
│    │                                │
│    ▼                                │
│  资源清理/状态持久化                 │
└─────────────────────────────────────┘
    │
    ▼
响应发送
```

### 4.1.2 Hook接口定义

```typescript
// Hook处理器接口
interface HookHandler<TContext = any> {
  (event: HookEvent<TContext>): Promise<void> | void;
}

// Hook事件结构
interface HookEvent<TContext = any> {
  readonly type: string;           // 事件类型（如 'agent', 'message'）
  readonly action: string;         // 事件动作（如 'bootstrap', 'run'）
  readonly context: TContext;      // 上下文对象（类型依赖具体Hook）
  readonly sessionKey?: string;    // 会话标识
  readonly timestamp: number;      // 事件时间戳
}

// Hook注册选项
interface HookRegistration {
  pluginId?: string;              // 注册的插件ID
  priority?: number;               // 执行优先级（数字越小越先执行）
  timeout?: number;               // 超时时间（ms）
  filter?: (event: HookEvent) => boolean;  // 条件过滤
}

// Hook管理器核心接口
interface HookManager {
  // 注册Hook
  register(
    type: string, 
    action: string, 
    handler: HookHandler,
    options?: HookRegistration
  ): string;  // 返回hookId
  
  // 注销Hook
  unregister(hookId: string): void;
  
  // 触发Hook
  emit(event: HookEvent): Promise<HookResult[]>;
  
  // 批量注册（用于Plugin）
  registerMany(hooks: HookDefinition[]): string[];
}
```

## 4.2 六类Hook深度解析

### 4.2.1 Start Hooks（启动钩子）

Start Hooks在Gateway或Session启动时触发，常用于初始化共享资源：

```typescript
// gateway.bootstrap：Gateway首次启动
export const gatewayBootstrapHook: HookHandler = async (event) => {
  if (event.type !== 'gateway' || event.action !== 'bootstrap') return;
  
  console.log('[Hook] Gateway bootstrapping...');
  
  // 初始化工作区
  await initializeWorkspace();
  
  // 加载持久化状态
  await loadPersistedState();
  
  // 注册全局错误处理器
  process.on('uncaughtException', handleGlobalError);
};

// session.start：每个新Session创建时
export const sessionStartHook: HookHandler = async (event) => {
  if (event.type !== 'session' || event.action !== 'start') return;
  
  const { sessionKey, context } = event;
  console.log(`[Hook] Session started: ${sessionKey}`);
  
  // 加载用户记忆
  await injectUserMemory(context);
  
  // 初始化Session状态
  context.state = {
    createdAt: Date.now(),
    messageCount: 0,
    activeTask: null
  };
};
```

### 4.2.2 Auth Hooks（认证钩子）

Auth Hooks在认证流程的关键节点拦截，用于权限校验：

```typescript
// auth.user：用户认证时
interface AuthHookContext {
  user: {
    id: string;
    identity: string;
    passport?: AgentPassport;
  };
  channel: ChannelId;
  credentials?: any;
  authResult: 'pending' | 'granted' | 'denied';
}

export const authUserHook: HookHandler = async (event: HookEvent<AuthHookContext>) => {
  if (event.type !== 'auth' || event.action !== 'user') return;
  
  const { user, channel } = event.context;
  
  // 检查IP白名单
  const allowedIPs = getConfig('security.allowedIPs');
  if (allowedIPs && !allowedIPs.includes(getClientIP())) {
    event.context.authResult = 'denied';
    throw new Error('IP not in whitelist');
  }
  
  // 检查用户权限等级
  const userLevel = await getUserLevel(user.id);
  const requiredLevel = getRequiredLevel(channel);
  
  if (userLevel < requiredLevel) {
    event.context.authResult = 'denied';
    throw new Error(`Insufficient permission level: ${requiredLevel}`);
  }
  
  event.context.authResult = 'granted';
};

// 认证级别枚举
enum AuthLevel {
  ANONYMOUS = 0,
  BASIC = 1,
  TRUSTED = 2,
  ELEVATED = 3,
  ADMIN = 4
}
```

### 4.2.3 Message Hooks（消息钩子）

Message Hooks拦截消息的接收和发送过程：

```typescript
// message.received：收到用户消息时
export const messageReceivedHook: HookHandler = async (event) => {
  if (event.type !== 'message' || event.action !== 'received') return;
  
  const { content, from } = event.context.message;
  
  // 内容安全检查
  const safetyResult = await checkContentSafety(content.text);
  if (!safetyResult.safe) {
    event.context.blocked = true;
    event.context.blockReason = safetyResult.reason;
    return;
  }
  
  // 敏感词过滤
  content.text = filterSensitiveWords(content.text);
  
  // 消息去重（基于内容hash）
  const msgHash = hashMessage(content);
  if (await isDuplicateMessage(msgHash)) {
    event.context.dropped = true;
  }
};

// message.parse：消息解析时
export const messageParseHook: HookHandler = async (event) => {
  if (event.type !== 'message' || event.action !== 'parse') return;
  
  const { raw, parsed } = event.context;
  
  // 指令检测
  const commandMatch = raw.text?.match(/^\\/(\\w+)(.*)$/);
  if (commandMatch) {
    parsed.command = {
      name: commandMatch[1],
      args: commandMatch[2]?.trim() || ''
    };
    parsed.type = 'command';
  }
  
  // 提及检测
  const mentionPattern = /@(\\w+)/g;
  const mentions = [...raw.text.matchAll(mentionPattern)];
  if (mentions.length > 0) {
    parsed.mentions = mentions.map(m => m[1]);
    parsed.type = 'mention';
  }
};

// message.outbound：发送消息前
export const messageOutboundHook: HookHandler = async (event) => {
  if (event.type !== 'message' || event.action !== 'outbound') return;
  
  const { content, target } = event.context;
  
  // 消息太长则自动摘要
  if (content.text?.length > 4000) {
    content.text = await summarizeLongMessage(content.text);
  }
  
  // 添加footer标记
  content.text += '\\n\\n---\\nSent via OpenClaw';
};
```

### 4.2.4 ToolUse Hooks（工具使用钩子）

ToolUse Hooks是最常用的Hook类型，用于监控和干预工具执行：

```typescript
// tool.use.pre：工具调用前
export const toolUsePreHook: HookHandler = async (event) => {
  if (event.type !== 'tool' || event.action !== 'use.pre') return;
  
  const { tool, args, sessionKey } = event.context;
  
  console.log(`[Hook] Tool call: ${tool} with args:`, args);
  
  // 工具使用日志
  await logToolUsage({
    tool,
    args: sanitizeArgs(args),  // 脱敏敏感信息
    sessionKey,
    timestamp: Date.now()
  });
  
  // 危险工具二次确认
  const dangerousTools = ['exec', 'rm', 'format'];
  if (dangerousTools.includes(tool)) {
    if (!await confirmDangerousOperation(tool, args)) {
      throw new Error('Operation cancelled by user');
    }
  }
  
  // 预算检查
  const budget = await getSessionBudget(sessionKey);
  if (budget.exhausted) {
    throw new Error('Session budget exhausted');
  }
};

// tool.use.post：工具调用后
export const toolUsePostHook: HookHandler = async (event) => {
  if (event.type !== 'tool' || event.action !== 'use.post') return;
  
  const { tool, args, result, duration, error } = event.context;
  
  // 成功日志
  if (!error) {
    console.log(`[Hook] Tool ${tool} completed in ${duration}ms`);
    await recordToolSuccess({ tool, duration });
  } else {
    // 错误处理
    console.error(`[Hook] Tool ${tool} failed:`, error);
    
    // 错误分类
    const errorType = classifyError(error);
    
    // 自动重试判断
    if (errorType === 'transient' && event.context.retries < 3) {
      event.context.shouldRetry = true;
      event.context.retryDelay = calculateBackoff(event.context.retries);
    }
    
    // 错误学习（self-improving-agent）
    if (errorType === 'user_correctable') {
      await notifyLearningSystem({
        type: 'error',
        tool,
        error: error.message,
        context: args
      });
    }
  }
};

// 错误分类
function classifyError(error: Error): 'transient' | 'permanent' | 'user_correctable' {
  const transientPatterns = [
    'ECONNREFUSED', 'ETIMEDOUT', 'ENOTFOUND',
    'rate limit', 'timeout', 'unavailable'
  ];
  
  const userCorrectablePatterns = [
    'syntax error', 'invalid argument', 'permission denied'
  ];
  
  for (const pattern of transientPatterns) {
    if (error.message.includes(pattern)) return 'transient';
  }
  for (const pattern of userCorrectablePatterns) {
    if (error.message.includes(pattern)) return 'user_correctable';
  }
  return 'permanent';
}
```

### 4.2.5 PromptSubmit Hooks（Prompt提交钩子）

PromptSubmit Hooks在Prompt发送给模型前拦截，允许动态修改：

```typescript
// prompt.submit：Prompt提交给LLM前
export const promptSubmitHook: HookHandler = async (event) => {
  if (event.type !== 'prompt' || event.action !== 'submit') return;
  
  const { prompt, context, session } = event.context;
  
  // 动态注入System Prompt片段
  if (shouldInjectContext(session)) {
    const dynamicContext = await buildDynamicContext(session);
    prompt.systemPrompt += '\\n\\n' + dynamicContext;
  }
  
  // Token预算管理
  const maxTokens = prompt.maxTokens || 4096;
  const usedTokens = estimatePromptTokens(prompt);
  
  if (usedTokens > maxTokens * 0.9) {
    // 触发上下文压缩
    prompt.messages = await compressContext(prompt.messages, maxTokens * 0.7);
    event.context.compressed = true;
  }
  
  // Persona增强
  const personaEnhancement = await buildPersonaEnhancement(session);
  prompt.systemPrompt += '\\n\\n' + personaEnhancement;
};

// UserPromptSubmit：用户Prompt提交时（另一个时机）
export const userPromptSubmitHook: HookHandler = async (event) => {
  if (event.type !== 'prompt' || event.action !== 'user_submit') return;
  
  const { userMessage, session } = event.context;
  
  // 任务延续检测
  if (isContinuationOfTask(userMessage, session)) {
    // 注入当前任务上下文
    event.context.hints = {
      ...event.context.hints,
      activeTask: session.state.activeTask,
      lastDecision: session.state.lastDecision
    };
  }
  
  // 学习信号检测
  if (containsLearningSignal(userMessage)) {
    await triggerLearningExtraction(userMessage);
  }
};

// 上下文压缩实现
async function compressContext(
  messages: ConversationMessage[], 
  targetTokens: number
): Promise<ConversationMessage[]> {
  const compressed: ConversationMessage[] = [];
  let currentTokens = 0;
  
  // 从最新消息开始，保留重要上下文
  for (let i = messages.length - 1; i >= 0; i--) {
    const msg = messages[i];
    const msgTokens = estimateTokens(msg);
    
    if (currentTokens + msgTokens <= targetTokens) {
      compressed.unshift(msg);
      currentTokens += msgTokens;
    } else if (msg.role === 'system') {
      // 系统消息必须保留
      compressed.unshift(msg);
    }
  }
  
  return compressed;
}
```

### 4.2.6 Stop Hooks（停止钩子）

```typescript
// agent.shutdown：Agent关闭时
export const agentShutdownHook: HookHandler = async (event) => {
  if (event.type !== 'agent' || event.action !== 'shutdown') return;
  
  const { session, reason } = event.context;
  
  console.log(`[Hook] Agent shutting down: ${reason}`);
  
  // 保存Session状态
  await persistSessionState(session);
  
  // 清理临时资源
  await cleanupTempResources(session);
  
  // 触发记忆保存
  if (shouldSaveMemory(session)) {
    await triggerMemorySave(session);
  }
};

// session.end：Session结束时
export const sessionEndHook: HookHandler = async (event) => {
  if (event.type !== 'session' || event.action !== 'end') return;
  
  const { sessionKey, context } = event;
  
  // 生成会话摘要
  const summary = await generateSessionSummary(context);
  
  // 更新记忆索引
  await updateMemoryIndex(sessionKey, summary);
  
  // 发送会话结束通知
  await notifySessionEnd(sessionKey, summary);
};
```

## 4.3 Hook注册机制

### 4.3.1 声明式注册（Plugin推荐方式）

```typescript
// Plugin中声明式注册Hook
export const pluginDefinition = {
  id: 'my-plugin',
  name: 'My Plugin',
  version: '1.0.0',
  
  hooks: [
    {
      type: 'tool',
      action: 'use.post',
      handler: 'onToolUsePost',
      priority: 100  // 较低优先级，后执行
    },
    {
      type: 'message',
      action: 'received',
      handler: 'onMessageReceived',
      filter: (event) => event.context.message.text?.length > 0
    }
  ],
  
  // Hook处理方法
  handlers: {
    async onToolUsePost(event: HookEvent) {
      // 实现
    },
    async onMessageReceived(event: HookEvent) {
      // 实现
    }
  }
};
```

### 4.3.2 运行时动态注册

```typescript
// 运行时动态注册Hook
const hookId = hookManager.register(
  'agent',
  'bootstrap',
  async (event) => {
    console.log('[MyHook] Agent bootstrapping!');
    
    // 注入虚拟引导文件
    if (Array.isArray(event.context.bootstrapFiles)) {
      event.context.bootstrapFiles.push({
        path: 'CUSTOM_BOOTSTRAP.md',
        content: '# Custom Bootstrap\\n\\nCustom initialization logic here.',
        virtual: true
      });
    }
  },
  {
    priority: 50,  // 高优先级，先执行
    timeout: 5000
  }
);

// 条件过滤示例
const conditionalHookId = hookManager.register(
  'message',
  'received',
  async (event) => {
    // 只处理包含特定关键词的消息
    if (event.context.message.content.text?.includes('[自动]')) {
      await handleAutoMessage(event);
    }
  },
  {
    filter: (event) => {
      // 过滤条件：只在测试环境启用
      return process.env.NODE_ENV === 'development';
    }
  }
);
```

### 4.3.3 Hook执行顺序

```typescript
// Hook执行顺序示例
interface HookExecutionPlan {
  hooks: {
    id: string;
    priority: number;
    handler: HookHandler;
  }[];
  totalTimeout: number;
}

// 按优先级排序执行
async function executeHooks(event: HookEvent, plan: HookExecutionPlan): Promise<void> {
  // 按优先级升序（数字越小越先执行）
  const sorted = [...plan.hooks].sort((a, b) => a.priority - b.priority);
  
  const results: HookResult[] = [];
  
  for (const hook of sorted) {
    try {
      const result = await Promise.race([
        hook.handler(event),
        new Promise((_, reject) => 
          setTimeout(() => reject(new Error('Hook timeout')), 5000)
        )
      ]);
      results.push({ hookId: hook.id, status: 'success', result });
    } catch (error) {
      results.push({ hookId: hook.id, status: 'error', error: error.message });
      
      // 根据错误处理策略决定是否继续
      if (isFatalError(error)) {
        throw error;  // 致命错误，停止执行
      }
    }
  }
  
  return results;
}
```

## 4.4 Hook与Skill的联动

Hook系统与Skill系统的联动是OpenClaw自动化的核心机制。

### 4.4.1 self-improving-agent的Hook集成

self-improving-agent是最典型的Hook+Skill联动案例：

```typescript
// self-improving-agent的Hook集成结构
class SelfImprovingAgent {
  private hooks: Map<string, HookHandler> = new Map();
  
  constructor() {
    this.registerHooks();
  }
  
  private registerHooks() {
    // PostToolUse Hook：捕获错误并学习
    this.hooks.set('tool-use-post', async (event) => {
      if (event.action !== 'use.post') return;
      
      const { tool, args, error, result } = event.context;
      
      if (error) {
        // 检测到错误，添加到学习队列
        await this.learnFromError(tool, error, args);
      } else {
        // 成功执行，检查是否需要记录最佳实践
        await this.checkForBestPractice(tool, args, result);
      }
    });
    
    // UserPromptSubmit Hook：任务后提醒评估学习
    this.hooks.set('user-prompt-submit', async (event) => {
      if (event.action !== 'user_submit') return;
      
      const session = event.context.session;
      
      // 检查是否有未保存的教训
      if (await this.hasUnprocessedLessons(session)) {
        // 注入提醒到Prompt
        event.context.hints = {
          ...event.context.hints,
          reminder: '你有一个未保存的教训，是否现在记录？'
        };
      }
    });
  }
  
  private async learnFromError(
    tool: string, 
    error: Error, 
    args: any
  ): Promise<void> {
    const entry = {
      tool,
      errorMessage: error.message,
      args: sanitizeForLearning(args),
      timestamp: Date.now(),
      occurrenceCount: 1
    };
    
    // 追加到ERRORS.md
    const errorsPath = '~/.self-improving/ERRORS.md';
    await appendToFile(errorsPath, formatErrorEntry(entry));
    
    // 检查是否达到推广阈值
    const existingErrors = await getErrorsForTool(tool);
    if (existingErrors.length >= 3) {
      await promoteToMemory(entry);
    }
  }
}
```

### 4.4.2 错误捕获Skill模板

基于Hook的错误捕获Skill通用模板：

```typescript
// error-catcher skill/hook integration
// SKILL.md
// ---
// name: error-catcher
// description: 自动捕获和记录错误
// ---

# Error Catcher Skill

## 功能

1. 监听ToolUse错误
2. 自动分类错误类型
3. 记录到错误日志
4. 尝试自动修复常见错误

## 工作流程

```
Tool执行失败 → Hook捕获 → 错误分类 → 记录日志 → (可选)自动修复
```

## 错误分类规则

| 错误类型 | 模式 | 处理策略 |
|----------|------|----------|
| 语法错误 | syntax/parse | 记录但不重试 |
| 权限错误 | permission | 通知用户 |
| 网络错误 | ECONN*/ETIMEDOUT | 重试3次 |
| 资源错误 | ENOENT/ENOTDIR | 尝试创建或通知 |
| 超时错误 | timeout | 重试或增加超时 |

## Hook集成代码

\`\`\`typescript
// scripts/error-catcher-hook.ts

// 错误分类函数
function classifyToolError(error: Error, tool: string): ErrorCategory {
  const msg = error.message.toLowerCase();
  
  if (msg.includes('syntax') || msg.includes('parse')) {
    return 'syntax';
  }
  if (msg.includes('permission') || msg.includes('denied')) {
    return 'permission';
  }
  if (msg.match(/econnrefused|etimedout|enotfound/i)) {
    return 'network';
  }
  if (msg.match(/enoent|enotdir/i)) {
    return 'resource';
  }
  if (msg.includes('timeout')) {
    return 'timeout';
  }
  return 'unknown';
}

// 自动修复映射
const autoFixStrategies: Record<ErrorCategory, (error: Error) => Promise<boolean>> = {
  'timeout': async (error) => {
    // 增加超时重试
    console.log('[ErrorCatcher] Retrying with increased timeout...');
    return true;
  },
  'resource': async (error) => {
    // 尝试创建缺失的目录
    const pathMatch = error.message.match(/'(.*?)'/);
    if (pathMatch) {
      const dir = pathMatch[1].split('/').slice(0, -1).join('/');
      await exec(`mkdir -p ${dir}`);
      return true;
    }
    return false;
  },
  'network': async (error) => {
    // 网络错误，等待后重试
    await new Promise(r => setTimeout(r, 2000));
    return true;
  },
  'syntax': async () => false,  // 无法自动修复
  'permission': async () => false,  // 需要用户介入
  'unknown': async () => false,
};

// 主Hook处理函数
export const postToolUseHook = async (event: HookEvent) => {
  if (event.type !== 'tool' || event.action !== 'use.post') return;
  
  const { tool, args, error, sessionKey } = event.context;
  
  if (!error) return;  // 只处理错误
  
  const category = classifyToolError(error, tool);
  
  // 记录错误日志
  await logError({
    tool,
    category,
    message: error.message,
    args: sanitizeArgs(args),
    sessionKey,
    timestamp: Date.now()
  });
  
  // 尝试自动修复
  if (category !== 'syntax' && category !== 'permission') {
    const fixed = await autoFixStrategies[category](error);
    if (fixed) {
      event.context.shouldRetry = true;
      event.context.retryDelay = 1000;
    }
  }
  
  // 发送通知
  if (category === 'permission') {
    await notifyUser(sessionKey, `权限错误：${error.message}`);
  }
};
\`\`\`

## 4.5 源码导读：Hook Manager实现

### 4.5.1 Hook Manager核心实现

```typescript
// HookManager核心实现（简化版）
class HookManagerImpl implements HookManager {
  private hooks: Map<string, HookRegistration> = new Map();
  private handlers: Map<string, HookHandler> = new Map();
  private executionOrder: string[] = [];
  
  register(
    type: string,
    action: string,
    handler: HookHandler,
    options: HookRegistration = {}
  ): string {
    const hookId = `${type}:${action}:${generateId()}`;
    
    this.hooks.set(hookId, {
      pluginId: options.pluginId,
      priority: options.priority || 100,
      timeout: options.timeout || 5000,
      filter: options.filter,
      type,
      action
    });
    
    this.handlers.set(hookId, handler);
    this.rebuildExecutionOrder();
    
    return hookId;
  }
  
  async emit(event: HookEvent): Promise<HookResult[]> {
    const matchingHooks = this.findMatchingHooks(event);
    const sorted = matchingHooks.sort(
      (a, b) => (this.hooks.get(a)?.priority || 100) - 
                (this.hooks.get(b)?.priority || 100)
    );
    
    const results: HookResult[] = [];
    
    for (const hookId of sorted) {
      const hook = this.hooks.get(hookId)!;
      const handler = this.handlers.get(hookId)!;
      
      try {
        const result = await this.executeWithTimeout(
          handler,
          event,
          hook.timeout
        );
        results.push({ hookId, status: 'success', result });
      } catch (error) {
        results.push({ 
          hookId, 
          status: 'error', 
          error: error.message 
        });
        
        if (this.isFatalError(error)) {
          throw error;
        }
      }
    }
    
    return results;
  }
  
  private findMatchingHooks(event: HookEvent): string[] {
    return [...this.hooks.entries()]
      .filter(([_, hook]) => {
        return hook.type === event.type && hook.action === event.action;
      })
      .filter(([_, hook]) => {
        return !hook.filter || hook.filter(event);
      })
      .map(([id, _]) => id);
  }
  
  private async executeWithTimeout(
    handler: HookHandler,
    event: HookEvent,
    timeout: number
  ): Promise<any> {
    return Promise.race([
      handler(event),
      new Promise((_, reject) => 
        setTimeout(() => reject(new Error('Hook timeout')), timeout)
      )
    ]);
  }
  
  private rebuildExecutionOrder(): void {
    this.executionOrder = [...this.hooks.entries()]
      .sort((a, b) => 
        (a[1].priority || 100) - (b[1].priority || 100)
      )
      .map(([id, _]) => id);
  }
  
  private isFatalError(error: Error): boolean {
    // 特定错误类型被认为是致命的
    return error.message.includes('[FATAL]');
  }
}
```

### 4.5.2 全局Hook实例

```typescript
// 全局Hook管理器单例
export const globalHookManager = new HookManagerImpl();

// 便捷方法
export const hooks = {
  onToolUse: (handler: HookHandler, options?: HookRegistration) =>
    globalHookManager.register('tool', 'use.*', handler, options),
    
  onMessage: (handler: HookHandler, options?: HookRegistration) =>
    globalHookManager.register('message', '.*', handler, options),
    
  onAgent: (handler: HookHandler, options?: HookRegistration) =>
    globalHookManager.register('agent', '.*', handler, options),
    
  onPrompt: (handler: HookHandler, options?: HookRegistration) =>
    globalHookManager.register('prompt', '.*', handler, options),
};
```

## 本章小结

本章深入剖析了OpenClaw的Hook系统：

1. **六大Hook类型**：Start/Stop/Auth/Message/ToolUse/PromptSubmit，覆盖Agent运行时的完整生命周期。

2. **触发时机图谱**：通过可视化时序图展示了各类型Hook在消息流中的精确位置。

3. **Hook接口定义**：标准化的`HookHandler`接口和`HookEvent`结构保证了扩展性。

4. **注册机制**：支持声明式注册（Plugin）和运行时动态注册两种方式。

5. **执行顺序**：通过`priority`控制执行顺序，通过`filter`实现条件触发。

6. **Hook与Skill联动**：self-improving-agent的案例展示了如何用Hook驱动Skill的自动学习。

7. **源码导读**：分析了HookManager的核心实现逻辑。

掌握Hook系统后，你能够在OpenClaw运行时任何关键节点插入自定义逻辑，实现高度定制化的自动化能力。下一章我们将学习Plugin系统，了解如何构建完整的OpenClaw扩展插件。
