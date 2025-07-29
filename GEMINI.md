# A2A (Agent-to-Agent) 协议详细总结

## 1. 简介

A2A (Agent-to-Agent) 协议是一个开放标准，旨在促进独立、异构的 AI 代理（Agent）系统之间的通信和互操作性。它提供了一套通用的语言和交互模型，使得由不同技术、不同厂商构建的代理能够相互发现、协作并完成复杂任务。

### 1.1 核心目标

- **互操作性**: 连接不同的代理系统，打破通信壁垒。
- **协作**: 支持代理间的任务委托、上下文交换和协同工作。
- **发现**: 允许代理动态地发现和理解其他代理的能力。
- **灵活性**: 支持同步、流式和异步推送等多种交互模式。
- **安全性**: 采用成熟的企业级 Web 安全实践。
- **异步优先**: 原生支持可能包含人工介入的长时间运行任务。

### 1.2 设计原则

- **简单**: 复用广泛应用的现有标准（HTTP, JSON-RPC 2.0, Server-Sent Events）。
- **企业级**: 与现有企业实践对齐，解决认证、授权、安全、隐私和监控等问题。
- **不透明执行**: 代理间的协作基于声明的能力和交换的信息，无需关心对方的内部实现。

## 2. 核心概念

- **A2A 客户端 (Client)**: 代表用户或系统发起请求的应用程序或代理。
- **A2A 服务端 (Server)**: 暴露 A2A 兼容端点，处理任务并提供响应的代理。
- **代理卡 (Agent Card)**: 由服务端发布的 JSON 元数据，描述其身份、能力、技能、服务端点和认证要求。
- **任务 (Task)**: A2A 管理工作的基本单元，拥有唯一 ID 和生命周期状态。
- **消息 (Message)**: 客户端和代理之间的一次通信交互，包含一个或多个内容部分（Part）。
- **内容部分 (Part)**: 消息或工件（Artifact）中的最小内容单元，可以是文本（`TextPart`）、文件（`FilePart`）或结构化数据（`DataPart`）。
- **工件 (Artifact)**: 代理执行任务所产生的输出，如文档、图片或数据。
- **流式传输 (Streaming)**: 使用 Server-Sent Events (SSE) 技术，实时增量地传递任务状态和结果。
- **推送通知 (Push Notifications)**: 对于长时间运行的任务，服务端通过 webhook 向客户端异步推送更新。

## 3. 传输与格式

- **传输协议**: 强制使用 **HTTP/HTTPS**。
- **数据格式**: 所有请求和响应的载荷（Payload）均使用 **JSON-RPC 2.0** 格式。`Content-Type` 头必须为 `application/json`。
- **流式传输**: 对于流式方法（如 `message/stream`），服务端响应的 `Content-Type` 为 `text/event-stream`，每个事件的 `data` 字段包含一个完整的 JSON-RPC 2.0 响应对象。

## 4. 代理发现：代理卡 (Agent Card)

代理卡是代理的“名片”，一个 JSON 文档，用于自我描述。

### 4.1 发现机制

- **Well-Known URI**: 推荐的标准地址 `https://{server_domain}/.well-known/agent.json`。
- **注册中心/目录**: 从企业或公共的代理注册中心查询。
- **直接配置**: 客户端预先配置好代理卡的 URL 或内容。

### 4.2 `AgentCard` 结构

代理卡包含以下关键信息：

- `protocolVersion`: A2A 协议版本。
- `name`, `description`, `provider`: 代理的名称、描述和提供商信息。
- `url`: 代理服务的首选端点 URL。
- `additionalInterfaces`: 支持的其他传输接口（如 gRPC, REST）。
- `capabilities`: 声明支持的可选功能，如 `streaming`、`pushNotifications`。
- `securitySchemes`, `security`: 描述访问端点所需的认证方案（如 OAuth 2.0, API Key），遵循 OpenAPI 规范。
- `skills`: 代理能够执行的具体能力列表，每个 skill 包含 ID、名称、描述、输入输出模式（MIME 类型）和示例。
- `defaultInputModes`, `defaultOutputModes`: 代理默认支持的输入输出 MIME 类型。

## 5. 核心数据对象

协议定义了一系列在 RPC 调用中交换的数据结构，主要包括：

- **`Task`**: 任务对象，包含 `id`, `contextId`, `status` (任务状态), `artifacts` (工件) 和 `history` (历史消息)。
- **`TaskState`**: 任务状态的枚举，如 `submitted`, `working`, `completed`, `failed`, `input-required` 等。
- **`Message`**: 消息对象，包含 `messageId`, `role` (`user` 或 `agent`), `parts` (内容部分) 等。
- **`Part`**: 内容部分，是一个联合类型，具体可以是：
    - **`TextPart`**: 包含纯文本。
    - **`FilePart`**: 包含文件，文件内容可以通过 URI (`FileWithUri`) 或 Base64 编码的字节 (`FileWithBytes`) 提供。
    - **`DataPart`**: 包含结构化的 JSON 数据。
- **`Artifact`**: 工件对象，任务的产出物，由一个或多个 `Part` 组成。
- **`PushNotificationConfig`**: 推送通知的配置，包含 `url` (webhook 地址) 和认证信息。

这些数据结构在 `specification/json/a2a.json` (JSON Schema) 和 `specification/grpc/a2a.proto` (Protobuf) 中有严格定义。

## 6. RPC 方法

A2A 协议定义了一套 RPC 方法，客户端通过向服务端的 `url` 发送 HTTP POST 请求来调用。

### 6.1 主要方法

| 方法名 | 描述 | 主要参数 | 返回值 |
| --- | --- | --- | --- |
| **`message/send`** | 发送消息以启动或继续一个任务。适用于同步或轮询场景。 | `MessageSendParams` (包含 `Message` 对象) | `Task` 或 `Message` |
| **`message/stream`** | 发送消息并订阅任务的实时更新。 | `MessageSendParams` | SSE 流，每个事件包含一个 `SendStreamingMessageResponse` |
| **`tasks/get`** | 获取指定任务的当前状态。 | `TaskQueryParams` (包含 `id`) | `Task` |
| **`tasks/cancel`** | 请求取消一个正在进行的任务。 | `TaskIdParams` (包含 `id`) | `Task` |
| **`tasks/resubscribe`** | 重新连接到某个任务的 SSE 流。 | `TaskIdParams` (包含 `id`) | SSE 流 |
| **`tasks/pushNotificationConfig/set`** | 为任务设置推送通知配置。 | `TaskPushNotificationConfig` | `TaskPushNotificationConfig` |
| **`agent/authenticatedExtendedCard`** | 获取认证后可能更详细的代理卡。 | 无 | `AgentCard` |

### 6.2 多协议支持

除了核心的 JSON-RPC，规范也为 gRPC 和 REST 定义了等效的接口，方便不同技术栈的系统集成。例如：
- JSON-RPC 的 `message/send` 对应 gRPC 的 `SendMessage` RPC 和 REST 的 `POST /v1/message:send`。
- `tasks/get` 对应 gRPC 的 `GetTask` 和 REST 的 `GET /v1/tasks/{id}`。

## 7. 错误处理

A2A 采用标准的 JSON-RPC 2.0 错误结构。除了标准错误码（如 `-32700` 解析错误, `-32601` 方法未找到），还定义了一系列 A2A 特定的错误码（在 `-32000` 到 `-32099` 范围内）：

| Code | 名称 | 描述 |
| --- | --- | --- |
| `-32001` | `TaskNotFoundError` | 找不到指定的任务 ID。 |
| `-32002` | `TaskNotCancelableError` | 任务当前状态不可取消。 |
| `-32003` | `PushNotificationNotSupportedError` | 代理不支持推送通知。 |
| `-32005` | `ContentTypeNotSupportedError` | 请求的内容类型不被支持。 |

## 8. 总结

A2A 协议通过一套定义明确的规范，为构建一个可互操作的、分布式的 AI 代理生态系统奠定了基础。它借鉴了成熟的 Web 技术和设计模式，在保证简单性和灵活性的同时，也充分考虑了企业应用场景下的安全和异步需求。开发者可以通过 `AgentCard` 发现代理能力，利用 `Task` 和 `Message` 进行多模态、多轮次的交互，并根据场景选择同步、流式或异步推送的方式来管理任务，从而实现强大而灵活的代理间协作。
