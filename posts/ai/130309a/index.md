# MCP架构和协议解析


{{&lt; admonition type=info title=&#34;内容简介&#34; open=true &gt;}}
介绍 **MCP** 的架构和协议实现。
{{&lt; /admonition &gt;}}

随着最近 MCP 在各类文章和评论区中逐渐活跃，我也找了几篇 MCP 相关的文章进行阅读学习，但是我发现这些文章都大同小异，没有让我真正深入理解MCP。我也跟着一些文章用 go 语言实现了几个 MCP 的Demo服务，但是对 MCP 依然一知半解，所以感觉还是得系统深入的学习并总结下。

## MCP是什么

MCP 是一种开放协议，用于标准化应用程序如何为 LLM 提供上下文。可以将 MCP 视为 AI 应用程序的 USB-C 端口。正如 USB-C 提供了一种标准化的方式来将设备连接到各种外围设备和配件，MCP 也提供了一种标准化的方式，用于将 AI 模型连接到不同的数据源和工具。

从上面这段官方的介绍来看，MCP 的目标是创建一个通用标准，使 AI 应用程序的开发和集成变得更加简单和统一。这里我引用一张流传很广的图片来帮助理解：
{{&lt; figure src=&#34;https://images.benelin.site/posts/mcp-architecture.jpg&#34; width=650 height=650 &gt;}}

## 为什么要用MCP

我们使用LLM的方式基本都是在客户端将问题发给大模型，然后大模型返回结果。面对一些特定的问题，还需要人工从数据库中筛选或者使用工具检索可能需要的信息，手动的粘贴到 prompt 中。但是随着问题一旦变得复杂，手动的粘贴信息到 prompt 中，就会非常麻烦。虽然许多 LLM 平台引入了 function call 功能。这一机制允许模型在需要时调用预定义的函数来获取数据或执行操作。但是 function call 有一个很明显的局限性，那就是不同 LLM 平台的 function call API 实现差异较大。开发者在切换模型时需要重写代码，适配成本很高。下面是一些 LLM 平台 function call API 的实现示例：

OpenAI:
```json
{
  &#34;index&#34;: 0,
  &#34;message&#34;: {
    &#34;role&#34;: &#34;assistant&#34;,
    &#34;content&#34;: null,
    &#34;tool_calls&#34;: [
      {
        &#34;name&#34;: &#34;get_current_stock_price&#34;,
        &#34;arguments&#34;: &#34;{\n  \&#34;company\&#34;: \&#34;Apple Inc.\&#34;,\n  \&#34;format\&#34;: \&#34;USD\&#34;\n}&#34;
      }
    ]
  },
  &#34;finish_reason&#34;: &#34;tool_calls&#34;
}
```
Claude:
```json
{
  &#34;role&#34;: &#34;assistant&#34;,
  &#34;content&#34;: [
    {
      &#34;type&#34;: &#34;text&#34;,
      &#34;text&#34;: &#34;&lt;thinking&gt;To answer this question, I will: …&lt;/thinking&gt;&#34;
    },
    {
      &#34;type&#34;: &#34;tool_use&#34;,
      &#34;id&#34;: &#34;toolu_01A09q90qw90lq917835lq9&#34;,
      &#34;name&#34;: &#34;get_current_stock_price&#34;,
      &#34;input&#34;: {&#34;company&#34;: &#34;Apple Inc.&#34;, &#34;format&#34;: &#34;USD&#34;}
    }
  ]
}
```
Gemini:
```json
{
  &#34;functionCall&#34;: {
    &#34;name&#34;: &#34;get_current_stock_price&#34;,
    &#34;args&#34;: {
        &#34;company&#34;: &#34;Apple Inc.&#34;,
        &#34;format&#34;: &#34;USD&#34;
    }
  }
}
```
从上面的示例也可以看到，各家 LLM 平台的 function call API有各自的调用格式。而 MCP 的出现统一了 AI 模型连接外部不同数据源和工具的标准，这也让各种各样的服务都开发了自己的 MCP 服务接口，进一步促进了 MCP 生态的开发和推广。

## MCP架构

### 核心架构解析

这里先引入官方的架构图：
{{&lt; figure src=&#34;https://images.benelin.site/posts/mcp-architecture2.jpg&#34; width=650 height=650 &gt;}}
MCP 由三个核心组件构成：Host、Client 和 Server。让我们通过一个实际场景来理解这些组件如何协同工作：假设你正在使用vscode的cline插件询问：&#34;我的main.go文件里面实现了什么功能？&#34;
1. Host：cline 作为 Host，负责接收你的提问并与你在cline中配置的模型交互。
2. Client：当大模型分析问题后发现需要读取你的main.go文件，Host 中内置的 MCP Client 会被激活。这个 Client 负责与适当的 MCP Server 建立连接。
3. Server：MCP Server 是一个特殊的应用程序，它负责与外部数据源或工具进行交互。在本例中，会使用 filesystem 这个 MCP Server 来读取 main.go 文件。

整个流程大致如下图所示：
```mermaid
graph LR
    subgraph 应用程序宿主进程
        H[主机]
        C1[客户端 1]
        C2[客户端 2]
        C3[客户端 3]
        H --&gt; C1
        H --&gt; C2
        H --&gt; C3
    end

    subgraph 本地机器
        S1[服务器 1&lt;br&gt;文件和 Git]
        S2[服务器 2&lt;br&gt;数据库]
        R1[(本地&lt;br&gt;资源 A)]
        R2[(本地&lt;br&gt;资源 B)]

        C1 --&gt; S1
        C2 --&gt; S2
        S1 &lt;--&gt; R1
        S2 &lt;--&gt; R2
    end

    subgraph 互联网
        S3[服务器 3&lt;br&gt;外部 API]
        R3[(远程&lt;br&gt;资源 C)]

        C3 --&gt; S3
        S3 &lt;--&gt; R3
    end
```
也就是说在MCP的工程实践中，Host和Client是集成在一起的，至于什么时候需要用到MCP Server，用哪一个MCP Server，这由大模型决定。而MCP server的功能由开发者根据需要去做不同的实现。

这种架构设计使得大模型可以在不同场景下灵活调用各种工具和数据源，而开发者只需专注于开发对应的 MCP Server，无需关心 Host 和 Client 的实现细节。

### 组件详情

**主机**
- 创建和管理多个客户端实例
- 控制客户端连接权限和生命周期
- 执行安全策略和同意要求
- 处理用户授权决策
- 协调 AI/LLM 集成和采样
- 管理跨客户端的上下文聚合

**客户端**

每个客户端由宿主创建，并维护一个隔离的服务器连接：
- 为每个服务器建立一个有状态会话
- 处理协议协商和能力交换
- 双向路由协议消息
- 管理订阅和通知
- 维护服务器之间的安全边界

一个宿主应用程序创建和管理多个客户端，每个客户端与特定的服务器有 1:1 的关系。

**服务器**

服务器提供专门的上下文和功能:

- 通过 MCP 原语公开资源、工具和提示
- 独立运行，具有专注的职责
- 通过客户端接口请求采样
- 必须尊重安全约束
- 可以是本地进程或远程服务

## 协议基础

MCP 中的所有消息必须遵循 [JSON-RPC 2.0](https://wiki.geekdream.com/Specification/json-rpc_2.0.html) 规范。协议定义了三种类型的消息：请求、响应和通知。

### 请求
双向消息，可以从客户端发送到服务器，也可以反向发送
- 必须包含字符串或整数类型的 ID
- ID 不能为 null
- 在同一会话中，请求方不能重复使用相同的 ID
- 可以包含可选的参数对象

请求示例：
```json
{
  &#34;jsonrpc&#34;: &#34;2.0&#34;,
  &#34;id&#34;: &#34;string | number&#34;,
  &#34;method&#34;: &#34;string&#34;,
  &#34;param?&#34;: {
    &#34;key&#34;: &#34;value&#34;
  }
}
```

### 响应
作为对请求的回复而发送。
- 必须包含与对应请求相同的 ID
- 必须设置 result 或 error 其中之一，不能同时设置
- 错误码必须是整数
- 可以包含可选的结果数据

响应示例：
```json
{
  &#34;jsonrpc&#34;: &#34;2.0&#34;,
  &#34;id&#34;: &#34;string | number&#34;,
  &#34;result?&#34;: {
    &#34;[key: string]&#34;: &#34;unknown&#34;
  },
  &#34;error?&#34;: {
    &#34;code&#34;: &#34;number&#34;,
    &#34;message&#34;: &#34;string&#34;,
    &#34;data?&#34;: &#34;unknown&#34;
  }
}
```

### 通知
不需要响应的单向消息，可以从客户端发送到服务器，也可以反向发送。
- 不能包含 ID 字段
- 用于状态更新和事件通知
- 可以包含可选的参数对象
- 减少通信开销，支持异步操作

通知示例：
```json
{
  &#34;jsonrpc&#34;: &#34;2.0&#34;,
  &#34;method&#34;: &#34;string&#34;,
  &#34;params?&#34;: {
    &#34;[key: string]&#34;: &#34;unknown&#34;
  }
}
```

## 生命周期
MCP 为客户端-服务器连接定义了严格的生命周期，确保正确的能力协商和状态管理。主要包含三个阶段：
1. 初始化：能力协商和协议版本协商
2. 操作：通过协议正常交换信息
3. 关闭：安全终止连接

生命周期时序图：
```mermaid
sequenceDiagram
    participant Client
    participant Server

    Note over Client,Server: 初始化阶段
    activate Client
    Client-&gt;&gt;&#43;Server: 初始化请求
    Server--&gt;&gt;Client: 初始化响应
    Client--)Server: 已初始化通知

    Note over Client,Server: 操作阶段
    rect rgb(200, 220, 250)
        note over Client,Server: 正常协议操作
    end

    Note over Client,Server: 关闭
    Client--)Server: 断开连接
    deactivate Server
    Note over Client,Server: 连接已关闭
```

### 初始化阶段

初始化阶段是客户端和服务器之间的第一次交互。在此阶段，双方：
- 建立协议版本兼容性
- 交换和协商能力
- 共享实现细节

其中的核心是版本协商和能力协商。
```mermaid
sequenceDiagram
    participant Host
    participant Client
    participant Server

    Host-&gt;&gt;&#43;Client: 初始化客户端
    Client-&gt;&gt;&#43;Server: 用能力初始化会话
    Server--&gt;&gt;Client: 响应支持的能力

    Note over Host,Server: 具有协商功能的活动会话

    loop 客户端请求
        Host-&gt;&gt;Client: 用户或模型发起的操作
        Client-&gt;&gt;Server: 请求（工具/资源）
        Server--&gt;&gt;Client: 响应
        Client--&gt;&gt;Host: 更新UI或响应模型
    end

    loop 服务器请求
        Server-&gt;&gt;Client: 请求（采样）
        Client-&gt;&gt;Host: 转发到AI
        Host--&gt;&gt;Client: AI响应
        Client--&gt;&gt;Server: 响应
    end

    loop 通知
        Server--)Client: 资源更新
        Client--)Server: 状态变更
    end

    Host-&gt;&gt;Client: 终止
    Client-&gt;&gt;-Server: 结束会话
    deactivate Server
```
### 操作阶段
在操作阶段，客户端和服务器根据协商的能力交换消息。
- 遵守协商的协议版本
- 仅使用成功协商的能力

### 关闭阶段
在关闭阶段，连接被优雅地终止。
- 客户端发送断开连接通知
- 服务器关闭连接
- 清理相关资源

## 传输机制

MCP 目前定义了两种标准的客户端-服务器通信传输机制：stdio（标准输入输出）和基于 SSE 的 HTTP。客户端应尽可能支持 stdio。此外，客户端和服务器也可以以可插拔的方式实现自定义传输机制。

### 标准输入输出（stdio）
在 stdio 传输中：
- 客户端将 MCP 服务器启动为子进程。
- 服务器在其标准输入（stdin）上接收 JSON-RPC 消息，并将响应写入其标准输出（stdout）。
- 消息由换行符分隔，并且不得包含嵌入的换行符。
- 服务器可以为日志记录目的向其标准错误（stderr）写入 UTF-8 字符串。客户端可以捕获、转发或忽略此日志记录。
- 服务器不得向其 stdout 写入任何不是有效 MCP 消息的内容。
- 客户端不得向服务器的 stdin 写入任何不是有效 MCP 消息的内容。
```mermaid
sequenceDiagram
    participant Client
    participant Server Process

    Client-&gt;&gt;&#43;Server Process: 启动子进程
    loop 消息交换
        Client-&gt;&gt;Server Process: 写入 stdin
        Server Process-&gt;&gt;Client: 写入 stdout
        Server Process--)Client: 可选的 stderr 日志
    end
    Client-&gt;&gt;Server Process: 关闭 stdin，终止子进程
    deactivate Server Process
```
### 基于 SSE 的 HTTP
在 SSE 传输中，服务器作为独立进程运行，可以处理多个客户端连接。服务器必须提供两个端点：
1. 一个 SSE 端点，供客户端建立连接并接收来自服务器的消息
2. 一个常规的 HTTP POST 端点，供客户端向服务器发送消息

当客户端连接时，服务器必须发送一个包含 URI 的 endpoint 事件，客户端用于发送消息。所有后续客户端消息必须作为 HTTP POST 请求发送到此端点。服务器消息作为 SSE message 事件发送，消息内容在事件数据中编码为 JSON。
```mermaid
sequenceDiagram
    participant Client
    participant Server

    Client-&gt;&gt;Server: 打开 SSE 连接
    Server-&gt;&gt;Client: endpoint 事件
    loop 消息交换
        Client-&gt;&gt;Server: HTTP POST 消息
        Server-&gt;&gt;Client: SSE message 事件
    end
    Client-&gt;&gt;Server: 关闭 SSE 连接
```
### 自定义传输
客户端和服务器可以以可插拔的方式实现自定义传输机制。该协议与传输无关，可以在任何支持双向消息交换的通信通道上实现。选择支持自定义传输的实施者必须确保他们保留 MCP 定义的 JSON-RPC 消息格式和生命周期要求。自定义传输应该记录其特定的连接建立和消息交换模式，以帮助互操作性。

---

> 作者: [beneliu](https://github.com/beneliu)  
> URL: https://beneliu.github.io/posts/ai/130309a/  

