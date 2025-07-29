# A2A over RabbitMQ: A Detailed Design Specification

- **作者**: Gemini
- **版本**: 1.0
- **日期**: 2025-07-23

## 1. 概述

本文档为在 RabbitMQ 消息队列上实现 Agent-to-Agent (A2A) 协议提供了一套详细的设计和实现规范。虽然 A2A 协议的核心规范以 HTTP + JSON-RPC 为基础，但其协议的语义和数据模型是传输独立的。本设计旨在利用消息队列的优势（如可靠性、异步性、可伸缩性），为构建更健壮的分布式 Agent 系统提供一个可行的蓝图。

本规范是一种**扩展实现**，它将 A2A 的 RPC 调用模式映射到 RabbitMQ 的消息传递模式上。

## 2. 核心概念：RPC over AMQP

我们将使用 RabbitMQ 的高级消息队列协议 (AMQP) 来模拟 A2A 的远程过程调用 (RPC)。其核心思想是：

- **请求 (Request)**: 客户端将一个 A2A 方法调用（如 `message/send`）封装成一条消息，并发送到一个指定的 Exchange。
- **响应 (Response)**: 服务端 Agent 处理请求后，将 A2A 的结果（如 `Task` 对象）封装成另一条消息，并发送回客户端指定的回复地址。
- **关联 (Correlation)**: 使用 `correlation_id` 属性来确保客户端能将收到的响应与它发出的请求正确匹配。

## 3. RabbitMQ 拓扑结构

为了支持 A2A 通信，我们定义以下 RabbitMQ 实体：

- **Exchange**: 一个名为 `a2a_exchange` 的 `topic` 类型的 Exchange。Topic Exchange 提供了最大的路由灵活性。
- **服务端请求队列 (Server's Request Queue)**: 每个 A2A Agent Server 实例监听一个持久化的、唯一的队列。命名规范为 `a2a.agent.[agent-name].requests`。
- **客户端回复队列 (Client's Reply Queue)**: 每个客户端实例在发起请求前，创建一个**临时的、排他的 (exclusive)** 队列，用于接收来自服务端的响应。

### 拓扑图

```mermaid
graph TD
    subgraph Client App
        C[Client] -- Publishes Request --> E
        C -- Consumes Reply --> RQ[Reply Queue (Temporary)]
    end

    subgraph RabbitMQ
        E(a2a_exchange<br/>type=topic)
    end

    subgraph Server Agent
        E -- Routing Key --> SQ[Server Queue<br/>a2a.agent.weather_bot.requests]
        S[Agent Server] -- Consumes Request --> SQ
        S -- Publishes Reply --> E
    end

    E -- reply_to Routing Key --> RQ
```

## 4. 协议映射：A2A 方法实现

### 4.1 同步请求/响应 (`message/send`)

这是最核心的交互模式，用于模拟标准的 RPC 调用。

#### 客户端实现步骤

1.  **创建回复队列**: 客户端为自身创建一个临时的、排他的回复队列。
2.  **生成关联 ID**: 客户端生成一个全局唯一的 `correlation_id` (e.g., UUID)。
3.  **构造并发布消息**:
    -   **Routing Key**: 指向目标 Agent 的请求队列，例如 `a2a.agent.weather_bot.request`。
    -   **Message Body**: A2A 方法的 `params` 对象，序列化为 JSON 字符串。例如 `MessageSendParams` 对象。
    -   **Message Properties**:
        -   `reply_to`: 设置为步骤 1 中创建的回复队列的名称。
        -   `correlation_id`: 设置为步骤 2 中生成的 ID。
        -   `content_type`: `application/json`。
        -   `headers`: `{'x-a2a-method': 'message/send'}` (自定义 Header，用于指定调用的方法)。
4.  **等待响应**: 客户端监听其回复队列，当收到 `correlation_id` 匹配的响应消息时，处理其 Body 作为 A2A 调用的结果。

#### 服务端实现步骤

1.  **监听请求队列**: 服务端 Agent 消费其主请求队列 (`a2a.agent.weather_bot.requests`)。
2.  **处理请求消息**:
    -   从消息属性中获取 `reply_to` 和 `correlation_id`。
    -   从 `headers` 中获取 `x-a2a-method`，确定要执行的 A2A 逻辑。
    -   反序列化消息 Body，执行业务逻辑（如创建 Task）。
3.  **构造并发送响应**:
    -   **Routing Key**: **必须**使用请求消息中的 `reply_to` 值。
    -   **Message Body**: A2A 方法的 `result` 对象（如 `Task` 对象），序列化为 JSON 字符串。
    -   **Message Properties**:
        -   `correlation_id`: **必须**将被请求消息的 `correlation_id` 复制过来。
        -   `content_type`: `application/json`。

### 4.2 流式传输 (`message/stream`)

利用消息队列的特性，可以非常自然地实现流式响应。

- **请求阶段**: 与同步模式完全相同，只是 `x-a2a-method` header 设置为 `message/stream`。
- **响应阶段**: 服务端**连续发送多条消息**到客户端的 `reply_to` 队列。每一条消息都使用相同的 `correlation_id`。
    -   每条消息的 Body 是一个 A2A 流式事件对象（如 `TaskStatusUpdateEvent` 或 `TaskArtifactUpdateEvent`）。
    -   **流结束信令**: 最后一条消息**必须**包含一个特殊的 header: `headers: {'x-a2a-stream-final': 'true'}`。客户端通过检查此 header 来判断流是否结束。

### 4.3 异步推送通知 (Push Notification)

这是消息队列相对于 Webhook 的最大优势所在，它更可靠且无需网络穿透。

1.  **客户端发起请求**:
    -   客户端调用 `message/send`，在其 `params.configuration.pushNotificationConfig` 中提供 MQ 的路由信息，而非 URL。
    -   **示例 `PushNotificationConfig`**:
        ```json
        {
          "transport": "rabbitmq",
          "routingKey": "client.user123.task_updates"
        }
        ```

2.  **服务端处理**: 服务端接收任务，并存储客户端提供的 `routingKey`。

3.  **服务端推送通知**: 当后台任务完成时，服务端将最终的 `Task` 对象作为消息体，发布到主 Exchange (`a2a_exchange`)，并使用客户端在步骤 1 中提供的 `routingKey`。

4.  **客户端接收**: 客户端只需监听一个绑定到其通知 `routingKey` 的队列，即可接收到任务完成的通知。

## 5. AgentCard 扩展规范

为了让其他 Agent 能够发现并使用基于 RabbitMQ 的 A2A 接口，`AgentCard` 必须进行如下扩展：

- 在 `additionalInterfaces` 数组中添加一个新条目。
- 使用 `metadata` 字段来承载 RabbitMQ 特定的连接信息。

### `AgentCard` 示例

```json
{
  "protocolVersion": "0.2.9",
  "name": "Financial Report Agent",
  "description": "An agent that generates complex financial reports asynchronously.",
  "url": "https://finance-agent.example.com/a2a/v1",
  "preferredTransport": "JSONRPC",
  "additionalInterfaces": [
    {
      "transport": "rabbitmq",
      "url": "amqps://user:pass@rabbitmq.example.com:5671/production",
      "metadata": {
        "exchange": "a2a_exchange",
        "requestRoutingKey": "a2a.agent.finance_report.requests",
        "notificationRoutingKeyPattern": "client.{clientId}.updates"
      }
    }
  ],
  "skills": [
    {
      "id": "generate-quarterly-report",
      "name": "Quarterly Report Generator",
      "description": "Generates a PDF report for a given quarter. This is a long-running task.",
      "outputModes": ["application/pdf"]
    }
  ]
}
```

- **`transport`**: 必须为 `rabbitmq`。
- **`url`**: 提供标准的 AMQP(S) 连接 URI。
- **`metadata.exchange`**: 指定用于 A2A 通信的 Exchange 名称。
- **`metadata.requestRoutingKey`**: 客户端向此 Agent 发送请求时应使用的路由键。
- **`metadata.notificationRoutingKeyPattern`**: (可选) 一个模式，用于指导客户端如何构造用于接收推送通知的路由键。

## 6. 消息结构总结

| 场景 | 消息 Body (Payload) | `reply_to` | `correlation_id` | `headers` |
| :--- | :--- | :--- | :--- | :--- |
| **Client Request** | A2A `params` object (JSON) | Client's temp queue | `unique-id-1` | `{'x-a2a-method': '...'}` |
| **Server Response** | A2A `result` object (JSON) | (Not set) | `unique-id-1` | (Optional) `{'x-a2a-stream-final': 'true'}` |
| **Server Push Notification** | Full `Task` object (JSON) | (Not set) | (Not applicable) | `{'x-a2a-notification': 'true'}` |

---

通过遵循本设计规范，开发者可以构建出利用消息队列强大能力的、完全兼容 A2A 协议语义的、高度可靠和可扩展的 AI Agent 系统。
