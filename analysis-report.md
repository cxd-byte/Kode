# Kode 项目架构深度分析报告 (附核心代码)

## 1. 项目概述

Kode 是一个功能强大的、基于终端的 AI 助手。它通过一个交互式的命令行界面（CLI），让用户能够与大语言模型（LLM）进行对话，并利用一系列专门的“Agent”和“工具”来完成复杂的软件开发任务，例如代码编写、文件操作、信息检索和任务自动化。

其核心设计理念是**模块化**和**可扩展性**。通过将不同的功能（如 Agent 定义、工具、上下文管理）分离到独立的模块中，系统保持了高度的灵活性和可维护性。开发者可以通过编写简单的 Markdown 文件来定义新的 Agent，或通过创建新的工具类来扩展系统的能力。

## 2. 核心技术栈

- **编程语言**: **TypeScript** (同时兼容 JavaScript)
- **运行环境**: **Node.js** (v20.18.1+)
- **开发/测试工具**: **Bun**
- **核心框架**:
    - **React & Ink**: 用于构建功能丰富的、交互式的命令行用户界面。
    - **Commander.js**: 负责解析命令行参数和注册子命令。
- **关键 AI/LLM 库**:
    - **`@anthropic-ai/sdk`**: 用于与 Anthropic Claude 系列模型进行交互。
    - **`openai`**: 用于与 OpenAI GPT 系列模型进行交互。
- **开发工具**:
    - **esbuild**: 用于快速打包和构建项目。
    - **Prettier & ESLint**: 用于保证代码风格统一和质量。

## 3. 目录结构与模块分析

### `src/entrypoints` - 应用入口

这是整个应用的起点。

- **`cli.tsx`**: 定义了应用的命令行接口，包括所有可用的命令（如 `kode`, `kode config`, `kode agents` 等）。它负责初始化环境、解析命令行参数，并根据用户的输入启动相应的模式——要么是直接执行一次性任务（`--print` 模式），要么是启动一个全功能的交互式会话（REPL）。
- **`mcp.ts`**: 启动 MCP (Model Context Protocol) 服务器的入口点，用于多 Agent 协作或连接外部服务。

### `src/commands` - 命令定义

此目录包含了用户可以在交互式会话中通过斜杠 `/` 调用的所有命令的实现。

- **`agents.tsx`**: 一个非常重要的命令，它提供了一个完整的终端 UI，用于管理 Agent 的生命周期（创建、查看、编辑、删除）。我们的分析主要从这里深入。
- 其他文件如 `bug.tsx`, `review.ts` 等，都定义了具体的、独立的命令功能。

### `src/utils/agentLoader.ts` - Agent 加载器

这是 Agent 系统的基石，负责从文件系统中发现、加载和管理 Agent 的定义。

- **Agent 定义**: Agent 是通过 `.md` (Markdown) 文件定义的。文件头部使用 YAML frontmatter 来定义元数据（名称、描述、可用工具等），文件的主体则是 Agent 的系统提示（System Prompt）。
- **分层覆盖系统**: Agent 的加载具有明确的优先级，实现了高度的灵活性。
- **热重载**: 该模块会监控所有 Agent 目录的文件变化。一旦有文件被修改，它会自动清除缓存，并在下次使用时重新加载，实现了 Agent 配置的动态更新，无需重启应用。

<details>
<summary>点击查看核心代码：Agent 加载与优先级覆盖</summary>

```typescript
// 文件: src/utils/agentLoader.ts

/**
 * 加载所有 Agent 配置
 */
async function loadAllAgents(): Promise<{
  activeAgents: AgentConfig[]
  allAgents: AgentConfig[]
}> {
  try {
    // 并行扫描所有可能的 Agent 目录
    const [userClaudeAgents, userKodeAgents, projectClaudeAgents, projectKodeAgents] = await Promise.all([
      scanAgentDirectory(join(homedir(), '.claude', 'agents'), 'user'),
      scanAgentDirectory(join(homedir(), '.kode', 'agents'), 'user'),
      scanAgentDirectory(join(getCwd(), '.claude', 'agents'), 'project'),
      scanAgentDirectory(join(getCwd(), '.kode', 'agents'), 'project')
    ])

    // 定义内置的默认 Agent
    const builtinAgents = [BUILTIN_GENERAL_PURPOSE]

    // 使用 Map 数据结构来实现优先级覆盖
    // 优先级顺序: built-in < user .claude < user .kode < project .claude < project .kode
    const agentMap = new Map<string, AgentConfig>()

    // 按照从低到高的优先级顺序将 Agent 添加到 Map 中
    // 如果遇到同名的 Agent，后加入的会自动覆盖前者。
    for (const agent of builtinAgents) {
      agentMap.set(agent.agentType, agent)
    }
    for (const agent of userClaudeAgents) {
      agentMap.set(agent.agentType, agent)
    }
    for (const agent of userKodeAgents) {
      agentMap.set(agent.agentType, agent)
    }
    for (const agent of projectClaudeAgents) {
      agentMap.set(agent.agentType, agent)
    }
    for (const agent of projectKodeAgents) {
      agentMap.set(agent.agentType, agent)
    }

    // 从 Map 中提取最终生效的 Agent 列表
    const activeAgents = Array.from(agentMap.values())

    return { activeAgents, allAgents: [...] }
  } catch (error) {
    // ... 错误处理
  }
}

// 使用 memoize (记忆化) 来缓存加载结果，提高性能
export const getActiveAgents = memoize(
  async (): Promise<AgentConfig[]> => {
    const { activeAgents } = await loadAllAgents()
    return activeAgents
  }
)
```
</details>

### `src/query.ts` - 核心交互循环

这是系统与大语言模型（LLM）交互的**大脑**。`query` 函数是整个请求-响应-工具使用的核心循环。

- **功能**: 它接收用户的输入、当前的消息历史、上下文信息以及一个可选的 `agentId`。
- **动态系统提示**: 它的关键功能是，如果收到了一个 `agentId`，它会动态加载该 Agent 的配置，并将其独特的系统提示（System Prompt）整合到发送给 LLM 的最终请求中。这使得 LLM 能够根据当前任务“扮演”一个特定的专家角色。
- **工具使用处理**: 当 LLM 返回一个使用工具的请求时，`query` 函数会负责解析这些请求，调用相应的工具，并将执行结果返回给 LLM，形成一个完整的“思考-行动”循环。

<details>
<summary>点击查看核心代码：动态应用 Agent 系统提示</summary>

```typescript
// 文件: src/query.ts

export async function* query(
  messages: Message[],
  systemPrompt: string[], // 基础的系统提示
  context: { [k: string]: string },
  canUseTool: CanUseToolFn,
  toolUseContext: ExtendedToolUseContext, // 包含了 agentId 等元数据
  // ...
): AsyncGenerator<Message, void> {

  // ...

  // 1. 根据传入的 agentId，格式化并生成最终的系统提示
  //    这是 Agent 个性化实现的最关键步骤。
  const { systemPrompt: fullSystemPrompt, reminders } =
    formatSystemPromptWithContext(systemPrompt, context, toolUseContext.agentId)

  // ...

  // 2. 调用 LLM，并将这个为 Agent 量身定制的系统提示传递过去
  function getAssistantResponse() {
    return queryLLM(
      normalizeMessagesForAPI(messages),
      fullSystemPrompt, // 使用包含 Agent 指令的完整系统提示
      // ...
    )
  }

  // 3. 发起请求并处理后续的工具调用...
  const result = await queryWithBinaryFeedback(
    toolUseContext,
    getAssistantResponse,
    // ...
  )

  // ...
}
```
</details>

### `src/utils/messageContextManager.ts` - 上下文管理器

这个模块扮演着“历史学家”和“档案管理员”的角色。它的主要职责是管理和压缩对话历史，确保发送给 LLM 的上下文不会超过其最大长度限制。

- **智能压缩**: 它提供了多种策略来处理长对话历史，例如将久远的消息智能地**总结**成一段摘要，从而在保留核心信息的同时节约宝贵的上下文空间。

<details>
<summary>点击查看核心代码：智能压缩策略</summary>

```typescript
// 文件: src/utils/messageContextManager.ts

export class MessageContextManager {
  /**
   * 策略3: 智能压缩，带总结摘要
   */
  private async smartCompressionStrategy(
    messages: Message[],
    strategy: MessageRetentionStrategy,
  ): Promise<MessageTruncationResult> {
    // 1. 将消息历史分为 "较旧" 和 "最近" 两部分
    const recentCount = Math.min(10, Math.floor(messages.length * 0.3))
    const recentMessages = messages.slice(-recentCount)
    const olderMessages = messages.slice(0, -recentCount)

    // 2. 为 "较旧" 的消息创建一个摘要
    const summary = this.createMessagesSummary(olderMessages)

    // 3. 创建一条新的 "系统消息"，内容是生成的摘要
    const summaryMessage: Message = {
      type: 'assistant',
      message: {
        role: 'assistant',
        content: [
          {
            type: 'text',
            text: `[CONVERSATION SUMMARY - ${olderMessages.length} messages compressed]\n\n${summary}\n\n[END SUMMARY - Recent context follows...]`,
          },
        ],
      },
      // ...
    }

    // 4. 返回由 [摘要] + [最近消息] 组成的新消息列表
    const truncatedMessages = [summaryMessage, ...recentMessages]

    return {
      truncatedMessages,
      removedCount: olderMessages.length,
      // ...
    }
  }

  // ... 辅助函数 createMessagesSummary 用于生成摘要内容
}
```
</details>

### `src/utils/toolExecutionController.ts` - 工具执行控制器

这是一个任务执行的“优化调度器”。当 LLM 请求同时使用多个工具时，这个控制器会介入，以最高效、最安全的方式来安排它们的执行。

- **并发与顺序**: 它会分析每个工具的属性（是否为只读操作），然后将工具调用分为可以并行执行的“并发组”和必须按顺序执行的“顺序组”。

<details>
<summary>点击查看核心代码：工具执行分组与调度</summary>

```typescript
// 文件: src/utils/toolExecutionController.ts

export class ToolExecutionController {
  private tools: Tool[]

  constructor(tools: Tool[]) {
    this.tools = tools
  }

  /**
   * 将一系列工具使用请求分组为并发和顺序执行组
   */
  groupToolsForExecution(
    toolUseMessages: ToolUseBlock[],
  ): ToolExecutionGroup[] {
    const groups: ToolExecutionGroup[] = []
    let currentGroup: ToolExecutionGroup = { concurrent: [], sequential: [] }

    for (const toolUse of toolUseMessages) {
      const tool = this.findTool(toolUse.name)

      // 1. 检查工具是否被标记为 "并发安全"
      if (tool && tool.isConcurrencySafe()) {
        // 2. 如果是，则将其添加到当前并发组
        currentGroup.concurrent.push(toolUse)
      } else {
        // 3. 如果不是 (例如，这是一个文件写入操作)，
        //    则先将之前的并发组保存起来，
        //    然后为这个不安全的工具创建一个新的、独立的顺序执行组。
        this.flushCurrentGroup(groups, currentGroup) // 保存旧组
        currentGroup = { concurrent: [], sequential: [toolUse] } // 创建新组
      }
    }

    this.flushCurrentGroup(groups, currentGroup) // 保存最后一组
    return groups
  }

  // ...
}
```
</details>

### `src/tools` - Agent 工具集

这个目录定义了所有 Agent 可以使用的“手”和“脚”。每一个子目录或文件都代表一个独立的工具。

- **工具定义**: 每个工具都定义了它的**名称**、**输入参数的模式（schema）**，以及最重要的 `call` 方法，该方法包含了工具执行的具体逻辑。
- **示例**: `FileReadTool/`, `BashTool/`, `WebSearchTool/` 等。

### `src/screens` - 交互界面

此目录包含了构成用户界面的核心 React/Ink 组件。

- **`REPL.tsx`**: Read-Eval-Print Loop，这是应用的主要交互界面。它维护着对话的状态，渲染消息历史，接收用户输入，并且是 **Agent 调度的发起者**。当用户输入特定指令（如 `@agent-name <任务>`）时，正是这个组件负责解析指令，并带着相应的 `agentId` 去调用核心的 `query` 函数。

<details>
<summary>点击查看核心代码：Agent 调度触发</summary>

```typescript
// 文件: src/utils/messages.ts (由 REPL.tsx 调用)

/**
 * 处理用户在 REPL 中输入的字符串
 */
export async function processUserInput(
  prompt: string,
  // ...
): Promise<MessageType[]> {

  // 1. 尝试从用户输入的开头提取 "@agent-name" 这样的标签
  const { tag: agentId, remainingPrompt } = extractTag(prompt)

  // 2. 如果成功提取出 agentId
  if (agentId) {
    // 2a. 验证这个 agent 是否真实存在
    const agent = await getAgentByType(agentId)
    if (agent) {
      // 2b. 创建一个特殊的用户消息，其中包含了要执行的任务
      //     这个消息随后会被 REPL.tsx 的 onQuery 处理器获取，
      //     并发起一个带有 agentId 的 query 调用。
      return [
        createUserMessage(
          `Okay, I will ask @${agentId} to: ${remainingPrompt}`,
          // ... options
        ),
      ]
    }
  }
  // ... 如果没有 @ 标签，则按正常流程处理
}

/**
 * 从字符串开头提取 @tag 的辅助函数
 */
export function extractTag(prompt: string): {
  tag: string | null
  remainingPrompt: string
} {
  // 使用正则表达式匹配 "@" 开头的单词
  const match = prompt.match(/^@([\w-]+)\s*/)
  if (match && match[1]) {
    return {
      tag: match[1], // agentId
      remainingPrompt: prompt.slice(match[0].length), // 剩余的任务描述
    }
  }
  return { tag: null, remainingPrompt: prompt }
}
```
</details>

## 4. Agent 架构与协作机制详解

Kode 的 Agent 系统采用了一种**用户驱动的、显式调度的协作模式**。它并非一个让多个 Agent 自主对话、自我组织的复杂系统，而是提供了一个强大的框架，让**用户**来扮演“项目经理”或“指挥中心”的角色。

**核心工作流程**:

1.  **加载 (Loading)**: 应用启动时，`agentLoader` 从文件系统加载所有可用的 Agent。
2.  **调度 (Dispatching)**: 用户在 `REPL` 界面中，通过**显式**的命令 `@agent-name <任务描述>` 来指定使用哪个 Agent 来处理接下来的任务。
3.  **角色扮演 (Role-Playing)**: `REPL` 组件捕获到这个指令后，会将对应的 `agentId` 传递给核心的 `query` 函数。`query` 函数随后加载该 Agent 独特的系统提示，并将其发送给 LLM。这使得 LLM 在接下来的交互中，会严格遵循该 Agent 的角色设定和行为指南。
4.  **执行 (Execution)**: “扮演”特定角色的 LLM 会根据任务需求，请求使用**被授权**的工具。这些请求被 `toolExecutionController` 高效地执行，结果返回给 LLM。
5.  **循环 (Looping)**: 这个过程会一直持续，直到任务完成。用户可以随时介入，或调用另一个不同的 Agent 来处理新的子任务。

---

## 5. 系统架构图 (Mermaid 语法)

```mermaid
graph TD
    subgraph "用户交互层 (User Interaction Layer)"
        U[用户 (User)]
        CLI(src/entrypoints/cli.tsx)
        REPL(src/screens/REPL.tsx)
    end

    subgraph "核心控制与编排层 (Core Control & Orchestration)"
        QUERY(src/query.ts)
        MCM(src/utils/messageContextManager.ts)
        TEC(src/utils/toolExecutionController.ts)
    end

    subgraph "数据与配置层 (Data & Configuration Layer)"
        AGENTS_MD(Agent .md 文件)
        AGENT_LOADER(src/utils/agentLoader.ts)
        TOOLS(src/tools/*)
    end

    subgraph "AI 与执行层 (AI & Execution Layer)"
        LLM[大语言模型 (Claude/OpenAI)]
        BASH[Bash/Terminal]
        FILESYS[文件系统 (Filesystem)]
        WEB[网络 (Web)]
    end

    %% -- 连接与数据流 --

    U -- "1. 输入指令 (@agent...)" --> REPL
    REPL -- "2. 解析 agentId, 调用 query(agentId)" --> QUERY

    AGENT_LOADER -- "加载/缓存" --> AGENTS_MD
    QUERY -- "3. 获取 Agent 定义(systemPrompt)" --> AGENT_LOADER

    MCM -- "4. 压缩对话历史" --> QUERY

    QUERY -- "5. 构建最终 Prompt" --> LLM
    LLM -- "6. 返回 tool_use 请求" --> QUERY

    QUERY -- "7. 提交工具执行计划" --> TEC
    TEC -- "8. 并发/顺序调度" --> TOOLS

    TOOLS -- "9. 执行具体操作" --> BASH
    TOOLS -- "9. 执行具体操作" --> FILESYS

    BASH -- "10. 返回结果" --> TOOLS
    FILESYS -- "10. 返回结果" --> TOOLS

    TOOLS -- "11. 格式化结果" --> QUERY
    QUERY -- "12. 将结果与历史再次发送" --> LLM

    LLM -- "13. 返回最终文本响应" --> QUERY
    QUERY -- "14. 将响应流式返回" --> REPL
    REPL -- "15. 渲染输出" --> U
```
