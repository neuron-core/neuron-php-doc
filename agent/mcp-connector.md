---
description: >-
  Connect the tools provided by Model Context Protocol (MCP) servers to your
  agent.
metaLinks:
  alternates:
    - https://app.gitbook.com/s/GHx4l2LknIex7vFIUg1R/agent/mcp-connector
---

# MCP

MCP (Model Context Protocol) is an open source standard designed by Anthropic to connect your agents to external service providers, such as your application database or external APIs.

Thanks to this protocol you can make tools exposed by an external server available to your agent.

Companies can build MCP servers to allow developers to connect Agents to their platforms. Here are a couple of directories with most used MCP servers:

* MCP official GitHub - [https://github.com/modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers)
* MCP-GET registry - [https://mcp-get.com/](https://mcp-get.com/)

### How it works

Neuron provides you with the `McpConnector` class that you can instantiate passing the MCP server configuration.

```php
use NeuronAI\MCP\McpConnector;

class MyAgent extends Agent 
{
    ...
    
    protected function tools(): array
    {
        return [
            ...McpConnector::make([
                'command' => 'php',
                'args' => ['/home/code/mcp_server.php'],
            ])->tools(),
        ];
    }
}
```

You should create an `McpConnector` instance for each MCP server you want to interact to.&#x20;

Neuron automatically discovers the tools exposed by the server and connects them to your agent.

When the agent decides to run a tool, Neuron will generate the appropriate request to call the tool on the MCP servers and return the result to the LLM to continue the task.  It feels exactly like with your own defined tools, but you can access a huge archive of predefined actions your agent can perform with just one line of code.

### Local MCP Server

If you want to connect with an MCP server installed locally on your machine or VM, you can use the "command" style configuration.

```php
use NeuronAI\MCP\McpConnector;

class MyAgent extends Agent 
{
    ...
    
    protected function tools(): array
    {
        return [
            ...McpConnector::make([
                'command' => 'php',
                'args' => ['/home/code/mcp_server.php'],
            ])->tools(),
        ];
    }
}
```

## Remote MCP Server

### Streamable HTTP Server

Remote servers are accessible via URLs and typically require authentication. You can use the `token` field in the configuration array, which will be used as the authorization token to authenticate on the server:

```php
use NeuronAI\MCP\McpConnector;

class MyAgent extends Agent 
{
    ...
    
    protected function tools(): array
    {
        return [
            ...McpConnector::make([
                'url' => 'https://mcp.example.com',
                'token' => 'BEARER_TOKEN',
                'timeout' => 30,
                'headers' => [
                    //'x-cutom-header' => 'value'
                ]
            ])->tools(),
        ];
    }
}
```

### SSE HTTP Transport

SSE ([Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)) is a mechanism that allows web clients to receive automatic updates from a server. Those updates are known as "events", and are sent over a single, long-lived HTTP connection.

To use the SSE transport you need to set `async ⇒ true` in the configuration parameters.

```php
use NeuronAI\MCP\McpConnector;

class MyAgent extends Agent 
{
    ...
    
    protected function tools(): array
    {
        return [
            ...McpConnector::make([
                'url' => 'https://mcp.example.com',
                'token' => 'BEARER_TOKEN',
                'timeout' => 30,
                'async' => true
            ])->tools(),
        ];
    }
}
```

## Monitoring & Debugging

Many of the applications you build with Neuron will contain multiple steps with multiple invocations of LLM calls. As these applications get more and more complex, it becomes crucial to be able to inspect what exactly is going on inside your agentic system. The best way to do this is with [Inspector](https://inspector.dev/).

{% embed url="https://docs.inspector.dev/guides/neuron-ai" %}

## Filter the list of tools

During connection with complex MCP servers they can includes tools that could lead to undesired behavior in specific contexts. The **`exclude()`** and **`only()`** methods address this challenge elegantly, allowing you to connect with large MCP servers while maintaining fine-grained control over available capabilities you want to provide to the agent.

These methods accept a list of tool names that you do, or do not want to associate with the agent.

```php
class MyAgent extends Agent 
{
    ...
    
    protected function tools()
    {
        return [
            // EXCLUDE: discard certain tools
            ...McpConnector::make([
                'url' => 'https://mcp.example.com',
            ])->exclude([
                'tool_name_1',
                'tool_name_2',
            ])->tools(),
            
            // ONLY: Select the tools you want to include
            ...McpConnector::make([
                'url' => 'https://mcp.example.com',
            ])->only([
                'tool_name_1',
                'tool_name_2',
            ])->tools(),
        ];
    }
}
```

### Interact with tool instances

If you need to apply specific policies to tools coming from an MCP server, like `requireApproval()` or `setMaxRuns()`, you can extract a tool instance using the **`with()`** method:

```php
class MyAgent extends Agent 
{
    ...
    
    protected function tools()
    {
        return [
            ...McpConnector::make([
                'url' => 'https://mcp.example.com',
            ])->with(
                name: 'tool_name_1',
                callback: fn (Tool $tool) => $tool->requireApproval()
            ])->tools(),
        ];
    }
}
```
