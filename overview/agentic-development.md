---
description: Connect the documentation to coding agents for AI Assisted Development
---

# Agentic Development

### Introduction

This documentation is also available and searchable as a Model Context Protocol (MCP) server.

This allows AI assistants to access Neuron AI documentation content directly, making it easy for tools like Claude Code, Cursor, and VS Code extensions reliably understand what Neuron AI can do, its components, how to configure them, and how to compose workflows agents and RAG.

The MCP server is available at: [https://docs.neuron-ai.dev/neuron-v2/\~gitbook/mcp](https://docs.neuron-ai.dev/neuron-v2/~gitbook/mcp)

### Claude Code

```
claude mcp add --transport http neuron-doc https://docs.neuron-ai.dev/neuron-v2/~gitbook/mcp
```

### VS Code

```json
"mcp": {
    "servers": {
        "neuron-ai-doc": {
            "type": "http",
            "url": "https://docs.neuron-ai.dev/neuron-v2/~gitbook/mcp"
        }
    }
}
```

### Cursor

```json
{
  "mcpServers": {
    "neuron-ai-doc": {
        "url": "https://docs.neuron-ai.dev/neuron-v2/~gitbook/mcp"
    }
  }
}
```

### Windsurf

```json
{
  "mcpServers": {
    "neuron-ai-doc": {
      "serverUrl": "https://docs.neuron-ai.dev/neuron-v2/~gitbook/mcp"
    }
  }
}
```

### OpenCode

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "neuron-ai-doc": {
      "type": "remote",
      "url": "https://docs.neuron-ai.dev/neuron-v2/~gitbook/mcp",
      "enabled": true
    }
  }
}
```
