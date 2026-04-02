# Claw Code 架构深度解析：AI Agent 系统设计的技术实践

> *AI 模型: minimax-m2.5 | 生成时间: 2026-04-02*
>
> *GitHub: https://github.com/ultraworkers/claw-code | 2026年3月开源，星标数 2 小时内突破 50K*

## 一、项目背景与定位

Claw Code 是一款面向开发者的高级 AI 编程 Agent，其核心定位是 **"Better Harness Tools, not merely storing the archive"**（更好的工具 harness，而非仅仅存储归档代码）。

该项目经历了重要的技术演进：

- **原始版本**：基于 TypeScript 的闭源实现
- **当前版本**：正在进行 Python → Rust 的重大迁移，采用系统级语言重写以获得更好的性能和内存安全

## 二、整体架构概览

```
rust/
├── crates/
│   ├── api/              # API 客户端（多提供商抽象）
│   ├── runtime/          # 核心运行时（会话、工具、权限、Hook）
│   ├── tools/            # 工具规范定义与执行
│   ├── commands/         # Slash 命令系统
│   ├── plugins/          # 插件系统
│   ├── claw-cli/         # CLI 入口
│   ├── server/           # HTTP/SSE 服务器
│   └── lsp/              # LSP 客户端集成
src/                      # Python 端口（进行中）
```

## 三、核心设计模式解析

### 3.1 对话运行时（Conversation Runtime）

这是整个系统的核心引擎，采用 **泛型 Trait 抽象**：

```rust
// 定义 API 客户端抽象接口
pub trait ApiClient {
    fn stream(&mut self, request: ApiRequest) -> Result<Vec<AssistantEvent>, RuntimeError>;
}

// 定义工具执行器抽象接口
pub trait ToolExecutor {
    fn execute(&mut self, tool_name: &str, input: &str) -> Result<String, ToolError>;
}

// 运行时组合这些抽象
pub struct ConversationRuntime<C, T> {
    session: Session,
    api_client: C,
    tool_executor: T,
    permission_policy: PermissionPolicy,
    // ...
}
```

**设计亮点**：

- 运行时与具体实现解耦，便于测试和替换
- 支持自定义 API 客户端和工具执行器

### 3.2 工具系统（Tool System）

工具系统采用 **三层架构**：

```
┌─────────────────────────────────────┐
│  GlobalToolRegistry                 │  ← 插件工具 + 内置工具聚合
├─────────────────────────────────────┤
│  mvp_tool_specs()                   │  ← 内置工具规范（bash, read_file, 等）
├─────────────────────────────────────┤
│  PluginTool                         │  ← 外部插件提供的工具
└─────────────────────────────────────┘
```

**内置工具示例**：

```rust
ToolSpec {
    name: "bash",
    description: "Execute a shell command in the current workspace.",
    input_schema: json!({
        "type": "object",
        "properties": {
            "command": { "type": "string" },
            "timeout": { "type": "integer", "minimum": 1 },
            "description": { "type": "string" },
        },
        "required": ["command"],
    }),
    required_permission: PermissionMode::DangerFullAccess,
}
```

### 3.3 权限模型（Permission Policy）

采用 **分级权限模型**，设计非常精细：

```rust
pub enum PermissionMode {
    ReadOnly,           // 只读
    WorkspaceWrite,     // 工作区写
    DangerFullAccess,   // 危险：完全访问
    Prompt,             // 每次询问
    Allow,              // 允许所有
}
```

**权限检查流程**：

```rust
pub fn authorize(&self, tool_name: &str, input: &str, prompter: Option<&mut dyn PermissionPrompter>) -> PermissionOutcome {
    // 1. 如果当前权限 >= 所需权限，直接放行
    if current_mode >= required_mode {
        return PermissionOutcome::Allow;
    }
    
    // 2. 需要提权时，询问 prompter
    if current_mode == PermissionMode::Prompt {
        return match prompter.decide(&request) {
            PermissionPromptDecision::Allow => PermissionOutcome::Allow,
            PermissionPromptDecision::Deny { reason } => PermissionOutcome::Deny { reason },
        };
    }
    
    // 3. 权限不足，拒绝
    PermissionOutcome::Deny { reason }
}
```

### 3.4 Hook 系统（运行时拦截）

支持 **PreToolUse** 和 **PostToolUse** 两种钩子：

```rust
pub enum HookEvent {
    PreToolUse,
    PostToolUse,
}

// Hook 执行结果
pub struct HookRunResult {
    denied: bool,
    messages: Vec<String>,
}

// 运行时集成
let pre_hook_result = self.hook_runner.run_pre_tool_use(&tool_name, &input);
if pre_hook_result.is_denied() {
    // 阻止工具执行
    return tool_result_with_error("PreToolUse hook denied");
}

let output = self.tool_executor.execute(&tool_name, &input)?;

let post_hook_result = self.hook_runner.run_post_tool_use(&tool_name, &input, &output, is_error);
// 可以修改输出或标记为错误
```

**配置方式**（通过 JSON）：

```json
{
  "hooks": {
    "PreToolUse": ["/path/to/pre-hook.sh"],
    "PostToolUse": ["/path/to/post-hook.sh"]
  }
}
```

### 3.5 多提供商支持（Provider Abstraction）

通过 Trait 抽象支持多个 AI 提供商：

```rust
pub trait Provider {
    type Stream;
    fn send_message(&self, request: &MessageRequest) -> ProviderFuture<MessageResponse>;
    fn stream_message(&self, request: &MessageRequest) -> ProviderFuture<Self::Stream>;
}

// 模型别名解析
pub fn resolve_model_alias(model: &str) -> String {
    match model.to_lowercase().as_str() {
        "opus" => "claude-opus-4-6".to_string(),
        "sonnet" => "claude-sonnet-4-6".to_string(),
        "grok" => "grok-3".to_string(),
        _ => model.to_string(),
    }
}

// 提供商自动检测
pub fn detect_provider_kind(model: &str) -> ProviderKind {
    if model.starts_with("grok") { ProviderKind::Xai }
    else if model.starts_with("claude") { ProviderKind::ClawApi }
    else { ProviderKind::OpenAi }
}
```

**当前支持的提供商**：

| 模型前缀 | 提供商 | 环境变量 |
|---------|--------|---------|
| claude-* | Claw API | ANTHROPIC_API_KEY |
| grok-* | xAI | XAI_API_KEY |
| gpt-* | OpenAI | OPENAI_API_KEY |

### 3.6 MCP 集成（Model Context Protocol）

完整的 MCP 协议支持，包括：

```rust
// MCP 服务器配置
pub enum McpServerConfig {
    Stdio(McpStdioServerConfig),      // 本地进程
    Sse(McpRemoteServerConfig),       // SSE 流
    Http(McpRemoteServerConfig),      // HTTP
    Ws(McpWebSocketServerConfig),     // WebSocket
    Sdk(McpSdkServerConfig),          // SDK 模式
    ManagedProxy(McpManagedProxyServerConfig),  // 托管代理
}

// 工具名称规范化
pub fn mcp_tool_name(server_name: &str, tool_name: &str) -> String {
    format!("mcp__{}__{}", normalize_name_for_mcp(server_name), normalize_name_for_mcp(tool_name))
}
```

### 3.7 会话压缩（Session Compaction）

当上下文接近 token 限制时，自动压缩历史会话：

```rust
pub fn compact_session(session: &Session, config: CompactionConfig) -> CompactionResult {
    // 1. 评估当前 token 使用量
    let estimated = estimate_session_tokens(session);
    
    // 2. 判断是否需要压缩
    if !should_compact(session, config) {
        return CompactionResult { /* 原样返回 */ };
    }
    
    // 3. 生成摘要（保留关键信息）
    let summary = summarize_messages(removed);
    
    // 4. 构造压缩后的会话
    let continuation = get_compact_continuation_message(&summary, true, !preserved.is_empty());
    
    // 5. 保留最近 N 条消息完整
    let compacted_messages = vec![
        ConversationMessage { role: MessageRole::System, blocks: vec![ContentBlock::Text { text: continuation }] },
        // ... 保留的消息
    ];
}
```

**压缩后的消息格式**：

```
<summary>
Conversation summary:
- Scope: 50 earlier messages compacted (user=20, assistant=20, tool=10).
- Tools mentioned: bash, read_file, write_file, grep.
- Recent user requests:
  - Implement the authentication flow
  - Add unit tests for user model
- Key timeline:
  - user: "I need to implement OAuth"
  - assistant: <tool_use name="read_file">...</tool_use>
  - tool: <tool_result>...</tool_result>
</summary>
```

### 3.8 插件系统（Plugin System）

支持动态扩展功能：

```rust
// 插件清单
pub struct PluginManifest {
    pub name: String,
    pub version: String,
    pub permissions: Vec<PluginPermission>,
    pub hooks: PluginHooks,
    pub lifecycle: PluginLifecycle,
    pub tools: Vec<PluginToolManifest>,
    pub commands: Vec<PluginCommandManifest>,
}

// 插件类型
pub enum PluginKind {
    Builtin,   // 内置插件
    Bundled,   // 捆绑插件
    External,  // 外部插件
}
```

## 四、Slash 命令系统

采用声明式命令注册：

```rust
const SLASH_COMMAND_SPECS: &[SlashCommandSpec] = &[
    SlashCommandSpec {
        name: "help",
        aliases: &[],
        summary: "Show available slash commands",
        category: SlashCommandCategory::Core,
    },
    SlashCommandSpec {
        name: "compact",
        summary: "Compact local session history",
        category: SlashCommandCategory::Core,
    },
    SlashCommandSpec {
        name: "branch",
        summary: "List, create, or switch git branches",
        category: SlashCommandCategory::Git,
    },
    // ... 20+ 命令
];
```

## 五、项目上下文发现

系统会自动发现项目根目录的配置文件：

```rust
fn discover_instruction_files(cwd: &Path) -> Vec<ContextFile> {
    // 向上遍历目录树，查找
    // - CLAW.md
    // - CLAW.local.md  
    // - .claw/CLAW.md
    // - .claw/instructions.md
}

pub fn discover_with_git(cwd: impl Into<PathBuf>, date: impl Into<String>) -> ProjectContext {
    let mut context = Self::discover(cwd, date)?;
    context.git_status = read_git_status(&context.cwd);
    context.git_diff = read_git_diff(&context.cwd);
    Ok(context)
}
```

## 六、关键设计启示

### 1. Trait 泛型解耦

```rust
// 运行时与具体实现完全解耦
impl<C, T> ConversationRuntime<C, T>
where
    C: ApiClient,
    T: ToolExecutor,
```

### 2. 分层配置系统

- ConfigLoader 支持多源配置（文件、环境变量）
- RuntimeConfig 运行时配置聚合
- 支持配置热重载

### 3. 事件驱动架构

- AssistantEvent 枚举所有可能的事件
- 支持流式处理（SSE）
- Hook 可拦截和修改行为

### 4. 权限安全模型

- 默认 `DangerFullAccess` 模式（安全风险意识）
- 可配置的工具级别权限要求
- 运行时权限提升需要确认

### 5. 错误处理

- 统一的错误类型定义
- 错误可恢复性（SessionError 支持 from 转换）
- 工具执行错误可被 Hook 拦截

## 七、与同类项目的对比

| 特性 | Claw Code | Claude Code | OpenCode |
|------|-----------|-------------|----------|
| 语言 | Rust (CLI) | TypeScript | Go |
| 多提供商 | ✓ | ✗ | ✗ |
| Hook 系统 | ✓ | ✓ | ? |
| 插件系统 | ✓ (进行中) | ✗ | ? |
| MCP 支持 | ✓ | ✓ | ? |
| 会话压缩 | ✓ | ✓ | ? |

## 八、总结

Claw Code 展示了构建企业级 AI Agent 系统的完整技术栈：

1. **运行时架构**：基于 Trait 的泛型设计，实现高度解耦
2. **安全模型**：分层的权限控制和 Hook 拦截机制
3. **扩展性**：插件系统和 MCP 协议支持
4. **工程实践**：Rust 实现的性能与内存安全保证
5. **开发者体验**：Slash 命令、CLAW.md 项目上下文、REPL 交互

该项目仍在活跃开发中（Python 端口和 Rust 重写并行），是理解现代 AI Agent 架构的绝佳参考。

---

*参考来源：https://github.com/ultraworkers/claw-code*
*本文档使用 mdBook 构建 — Rust 编写的极速静态网站生成器*