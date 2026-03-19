---
description: Connect the documentation to coding agents for AI Assisted Development
---

# Agentic Development

When working with AI coding assistants like Claude Code, Opencode, Cursor, or other similar tools, you can reference the Neuron AI documentation to give the AI deep context about our components. This leads to more accurate code suggestions, better understanding of component APIs, and fewer hallucinations when generating Neuron code.

## Agent Skills

The [Agent Skills specification](https://agentskills.io/) is a standard for providing structured documentation to AI coding assistants. It helps AI tools understand your project's APIs, conventions, and best practices through a well-organized directory of markdown files.

Neuron publishes an Agent Skill that provides AI tools with comprehensive information about our components, including their APIs, usage patterns, interfaces, and more.

### Accessing Skills

The Agent Skill is available in the Neuron AI vendor folder at:

```bash
vendor/neuron-core/neuron-ai/skills/
        └── neuron-agent-builder/
            └── SKILL.md
        └── neuron-debugger/
            └── SKILL.md
        └── neuron-rag-specialist/
            └── SKILL.md
        └── neuron-structured-output/
            └── SKILL.md
        └── neuron-tool-creator/
            └── SKILL.md
        └── neuron-test-engineer/
            └── SKILL.md
        └── neuron-tool-creator/
            └── SKILL.md
        └── neuron-workflow-architect/
            └── SKILL.md

```

### How to install skills

How you reference the skill depends on which AI tool you're using.

#### **Claude**

If you're using [Claude Code](https://claude.ai/code), you can install the Neuron AI skills locally using the [skills CLI](https://skills.sh/):

```bash
npx skills add ./vendor/neuron-core/neuron-ai/skills
```

Once installed, the skill will be available to Claude Code automatically. The skilla are installed as a symlink, so it will automatically stay up to date when you update Neuron via composer.

#### Cursor  <a href="#cursor" id="cursor"></a>

In [Cursor](https://cursor.sh/), you can add the skill directory to your project's documentation sources via **Cursor Settings > Features > Docs**. Point it to the `vendor/neuron-core/neuron-ai/skills` .

#### Other AI Tools  <a href="#other-ai-tools" id="other-ai-tools"></a>

Most AI coding assistants that support the Agent Skills specification can use this skill. Check your tool's documentation for how to add custom skills or documentation sources.

## MCP Server

This documentation is also available and searchable as a Model Context Protocol (MCP) server. This allows AI assistants to access Neuron AI documentation content directly. The MCP server is available at: [https://docs.neuron-ai.dev/\~gitbook/mcp](https://docs.neuron-ai.dev/~gitbook/mcp)

### Claude Code

```
claude mcp add --transport http neuron-ai-doc https://docs.neuron-ai.dev/~gitbook/mcp
```

### VS Code

```json
"mcp": {
    "servers": {
        "neuron-ai-doc": {
            "type": "http",
            "url": "https://docs.neuron-ai.dev/~gitbook/mcp"
        }
    }
}
```

### Cursor

```json
{
  "mcpServers": {
    "neuron-ai-doc": {
        "url": "https://docs.neuron-ai.dev/~gitbook/mcp"
    }
  }
}
```

### Windsurf

```json
{
  "mcpServers": {
    "neuron-ai-doc": {
      "serverUrl": "https://docs.neuron-ai.dev/~gitbook/mcp"
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
      "url": "https://docs.neuron-ai.dev/~gitbook/mcp",
      "enabled": true
    }
  }
}
```
