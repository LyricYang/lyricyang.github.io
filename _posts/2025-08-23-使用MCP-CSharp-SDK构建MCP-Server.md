---
layout:     post
title:      使用MCP C# SDK构建MCP Server
subtitle:   初识MCP工具
date:       2025-08-23 12:00:00
author:     AaronYeoh
header-img: img/mcp/post-bg-mcp.jpg
catalog: true
tags:
    - 技术分享
---

## 01｜引言

[模型上下文协议（MCP）](https://modelcontextprotocol.io/docs/getting-started/intro)是一种标准化协议，旨在通过提供一种结构化方式在大语言模型（LLMs）及其客户端之间交换上下文和数据。无论你是要构建人工智能驱动的应用，还是要将多个模型集成到一个聚合系统中，MCP 都能确保可操作性和可扩展性。

随着 [MCP C# SDK](https://github.com/modelcontextprotocol/csharp-sdk) 的发布，开发人员现在可以轻松构建利用该协议的服务器和客户端。该 SDK 简化了实施过程，使您可以专注于应用程序的独特功能，而不是复杂的协议处理。此外，SDK 还包括对 MCP 服务器消费的支持，使开发人员能够创建与 MCP 服务器无缝交互的强大客户端应用程序。

在 MCP 中，传输是指连接客户端和服务器并促进它们之间通信的机制。你可以根据运行 Server 的环境和需求选择最合适的传输方式。

1. stdio (Standard Input/Output)

stdio 是一种非常传统且常见的进程间通信（IPC）方式。这是一个双向、同步的流。连接在 Server 进程启动时建立，并在 Server 进程终止时关闭。无需网络，配置简单，每个会话启动独立的 Server 进程，相互隔离。

2. SSE (Server-Sent Events)

SSE 是一种基于 HTTP 的标准，允许服务器将更新主动“推送”到客户端。客户端向服务端的一个特定端点发起一个 HTTP GET 请求。这个连接会一直保持打开状态。服务端通过这个长连接，以流的形式持续发送事件（事件是符合特定文本格式的数据块）。

3. Streamable HTTP

Streamable HTTP是一种HTTP协议的流式传输技术，专门用于大文件的分段传输。与SSE不同，Streamable HTTP允许文件在传输的同时被处理，使客户端可以边接收数据边处理。

## 02｜构建MCP服务

MCP C# SDK以NuGet软件包的形式发布，你可以将其集成到一个网络应用服务中。

首先，安装NuGet软件包：

```shell
dotnet new web
dotnet add package ModelContextProtocol.AspNetCore --prerelease
```

### 构建MCP服务器

然后，通过更新Program.cs来创建我们的MCP服务，配置标准的服务传输，并告诉服务从程序集中搜索MCP工具（或可用的API）。

```c#
// Create a new WebApplication builder instance with command-line arguments
var builder = WebApplication.CreateBuilder(args);

// Register MCP server with HTTP transport
builder.Services.AddMcpServer()
    .WithHttpTransport()
    .WithToolsFromAssembly();
    .WithPromptsFromAssembly();

var app = builder.Build();
app.MapMcp();
app.Run();
```

`.WithToolsFromAssembly()`这个方法说明，服务在启动时会扫描程序集中带有`[McpServerToolType]`注解的类，以及用`[McpServerTool]`注解定义的MCP工具。

### 定义MCP工具（MCP Tools）

让我们定义一个最基本的工具：

```C#
[McpServerToolType]
public static class EchoTool
{
    [McpServerTool, Description("Echoes the message back to the client.")]
    public static string Echo([Description("Any message sent by the client")] string message) => $"hello {message}";
}
```

需要注意的是，MCPServerTool有一个描述（Description），它将提供工具说明给连接到服务器的客户端。该描述有助于客户端确定调用哪个工具。

输入参数message也有一个描述（Description），客户端也可以看到这个描述，有助于客户端调用工具时设置合适的参数。

### 定义MCP提示（MCP Prompts）

MCP Prompts提供可复用，可共享的预制指令模板。MCP Prompts 可以动态地接入外部数据和工具的结果，使提示词不再是静态文本。

使用`[MCPServerPrompt]`也能以类似的方式显示提示，例如：

```c#
[McpServerPromptType]
public static class MyPrompts
{
    [McpServerPrompt, Description("Creates a prompt to summarize the provided message.")]
    public static ChatMessage Summarize([Description("The content to summarize")] string content) =>
        new(ChatRole.User, $"Please summarize this content into a single sentence: {content}");
}
```

在启动代码中，`WithPromptsFromAssembly`将扫描程序集，查找带有`[McpServerPromptType]`属性的类，并注册带有`McpServerPrompt`属性的所有方法。与MCP工具类似，方法和参数都有一个描述。

## 03｜配置与运行

### MCP Inspector

我们可以在本地运行这个MCP服务，并通过[MCP Inspector](https://github.com/modelcontextprotocol/inspector)进行测试和调试。MCP Inspector Client（MCPI）是基于React的网络用户界面，为测试和调试MCP服务提供交互式界面。

要使用用户界面，需执行以下操作，并且需要安装Node.js: ^22.7.5.

```cmd
npx @modelcontextprotocol/inspector
```

默认情况下，MCP Inspector 代理服务器需要进行身份验证。启动服务器时，会生成一个随机会话令牌并打印到控制台：

```txt
🔑 Session token: 3a1c267fad21f7150b7d624c160b7f09b0b8c4f623c7107bbf13378f051538d4

🔗 Open inspector with token pre-filled:
   http://localhost:6274/?MCP_PROXY_AUTH_TOKEN=3a1c267fad21f7150b7d624c160b7f09b0b8c4f623c7107bbf13378f051538d4
```

对于对服务器的所有请求，此令牌必须作为Bearer令牌包含在授权标题中。检查器将自动打开您的浏览器，并在URL中预先填充了令牌。

![img](/img/mcp/Pasted_image_20250823163529.png)

Inspector提供多种与MCP服务器交互的功能，包括：Prompts和Tools。

### VS code - Github Copilot

要在本地运行我们的项目，并在.vscode文件夹或用户设置中的mcp.json文件中添加一个新服务器：

```json
"SFSPlatformServer": {
      "type": "sse",
      "url": "https://sfsplatformmcpserver.azurewebsites.net/sse",
      "alwaysAllow": []
    }
```

进入Github Copilot并切换到代理模式后，我们可以看到新添加的工具：

![img](/img/mcp/Pasted_image_20250823165525.png)

打开GitHub Copilot的“代理”模式，我们就可以提供一个prompt，它就会选择合适的工具进行调用。

![img](/img/mcp/Pasted_image_20250823165542.png)

