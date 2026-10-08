---
description: Easily implement LLM interactions with built-in memory and tool usage.
metaLinks:
  alternates:
    - https://app.gitbook.com/s/GHx4l2LknIex7vFIUg1R/agent/agent
---

# Agent

{% hint style="warning" %}
#### Coding Agent Skill

Use **`/neuron-agent`** to teach your coding agent how to implement an agent using Neuron AI components.

[AI-Assisted Development](../overview/agentic-development.md)
{% endhint %}

### Introduction

You can create your agent by extending the `NeuronAI\Agent\Agent` class to inherit the main features of the framework and create fully functional agents.&#x20;

This class automatically manages some mechanisms for you such as memory, tools and function calls. We will go into more detail about these aspects in the following sections.

We strongly encourage to extend the Agent class instead of creating agents using the [fluent definition](agent.md#fluent-agent-definition). This strategy make it easier to add custom methods and behaviour to the agent, and also promote portability, because all the moving parts are encapsulated into a single entity that you can run wherever you want in your application, or even release as a stand alone composer package.

Let's start creating an AI Agent summarizing YouTube videos. We start creating the `YouTubeAgent` class:

{% tabs %}
{% tab title="Unix" %}
```bash
vendor/bin/neuron make:agent App\\Neuron\\YouTubeAgent
```
{% endtab %}

{% tab title="Windows" %}
```powershell
.\vendor\bin\neuron make:agent App\Neuron\YouTubeAgent
```
{% endtab %}
{% endtabs %}

The command will create a class like this:

```php
<?php

namespace App\Neuron;

use NeuronAI\Agent\Agent;
use NeuronAI\Agent\SystemPrompt;
use NeuronAI\Providers\AIProviderInterface;

class YouTubeAgent extends Agent
{
    protected function provider(): AIProviderInterface
    {
        // return an instance of Anthropic, OpenAI, Gemini, Ollama, etc...
    }
    
    protected function instructions(): string
    {
        return new SystemMessage(
            "You are a friendly AI Agent created with Neuron framework."
        );
    }
    
    /**
     * @return \NeuronAI\Tools\ToolInterface[]
     */
    protected function tools(): array
    {
        return [];
    }
}
```

### Monitoring & Debugging

Many of the applications you build with Neuron will contain multiple steps with multiple invocations of LLM calls. As these applications get more and more complex, it becomes crucial to be able to inspect what exactly is going on inside your agentic system. The best way to do this is with [Inspector](https://inspector.dev/).

{% embed url="https://docs.inspector.dev/guides/neuron-ai" %}

### AI Provider

The minimum implementation requires assigning an AI Provider that will be the language and reasoning engine of your agent.

The only required method to implement is `provider()`  returning the instance of the provider you want to use. Let's assume it's Anthropic.

```php
<?php

namespace App\Neuron;

use NeuronAI\Agent\Agent;
use NeuronAI\Chat\Message\SystemMessage;
use NeuronAI\Providers\AIProviderInterface;
use NeuronAI\Providers\Anthropic\Anthropic;

class YouTubeAgent extends Agent
{
    protected function provider(): AIProviderInterface
    {
        // return an instance of Anthropic, OpenAI, Gemini, Ollama, etc...
        return new Anthropic(
            key: 'ANTHROPIC_API_KEY',
            model: 'ANTHROPIC_MODEL',
        );
    }
    
    protected function instructions(): SystemMessage
    {
        return new SystemMessage(
            "You are a friendly AI Agent created with Neuron framework."
        );
    }
    
    /**
     * @return \NeuronAI\Tools\ToolInterface[]
     */
    protected function tools(): array
    {
        return [];
    }
}
```

You can also use other providers like OpenAI, Gemini, or Ollama if you want to run the model locally. Check out the [supported providers](../providers/ai-provider.md).

### System instructions

The second important building block is the system instructions. System instructions provide directions for making the AI ​​act according to the task we want to achieve. They are fixed instructions that will be sent to the LLM on every interaction.

That’s why they are defined by an internal method, and stay encapsulated into the agent entity. Let's implement the `instructions()` method:

```php
<?php

namespace App\Neuron;

use NeuronAI\Agent\Agent;
use NeuronAI\Chat\Messages\SystemMessage;;
use NeuronAI\Providers\AIProviderInterface;
use NeuronAI\Providers\Anthropic\Anthropic;

class YouTubeAgent extends Agent
{
    protected function provider(): AIProviderInterface
    {
        // return an AI provider instance (Anthropic, OpenAI, Ollama, Gemini, etc.)
        return new Anthropic(
            key: 'ANTHROPIC_API_KEY',
            model: 'ANTHROPIC_MODEL',
        );
    }
    
    protected function instructions(): SystemMessage
    {
        return new SystemMessage(<<<TEXT
            You are an AI Agent specialized in writing YouTube video summaries.
            Get the url of a YouTube video, or ask the user to provide one.
            Use the tools you have available to retrieve the transcription of the video.
            Write a summary in a paragraph without using lists. Use just fluent text.
            After the summary add a list of three sentences as the three most important take away from the video.
        TEXT);
    }
    
    /**
     * @return \NeuronAI\Tools\ToolInterface[]
     */
    protected function tools(): array
    {
        return [];
    }
}
```

The `SystemMessage` class can be also populated with multiple content blocks `TextContent` in order to dynamically inject contents into the system instructions:

```php
class YouTubeAgent extends Agent
{
    protected function provider(): AIProviderInterface
    {
        ...
    }
    
    protected function instructions(): SystemMessage
    {
        $message = new SystemMessage(<<<TEXT
            You are an AI Agent specialized in writing YouTube video summaries.
            Get the url of a YouTube video, or ask the user to provide one.
            Use the tools you have available to retrieve the transcription of the video.
        TEXT);
        
        $message->addContent(
            new TextContent("Write a summary in a paragraph without using lists. Use just fluent text.")
        );
        
        $message->addContent(
            new TextContent("After the summary add a list of three sentences as the three most important take away from the video.")
        );
                
        return $message;
    }
}
```

If you are willing to use the system prompt caching for providers like Anthropic, OpenAI, Gemini, etc., you can call the `cache()` method on each content part you want to cache:

```php
$message->addContent(
    new TextContent("...")->cache()
);
```

Or call the cache method on the `SystemMessage` to cache the entire system prompt:

```php
    protected function instructions(): SystemMessage
    {
        $message = new SystemMessage(...);
        
        $message->addContent(
            new TextContent(...)
        );
        
        $message->addContent(
            new TextContent(...)
        );
                
        return $message->cache(); // <- cache everything
    }
```

### Dynamic Context and Cache

Part of your system instructions can be dynamic, or change frequently, like a date, some user information, etc.. You can supply this type of context through the `context()` hook or `setContext()`. Under the hood the Agent will inject this part at the end of the conversation, in order to optimize prompt cache on the provider API.&#x20;

Using context for the dynamic part of your system instructions you will probably see a drastical cost reduction.

```php
class SupportAgent extends Agent
{
    protected function instructions(): SystemMessage
    {
        return (new SystemMessage('You are a support agent...'))->cache(); // never changes
    }

    protected function context(): array
    {
        return [
            new TextContent('Today is ' . date('Y-m-d')),
            new TextContent('The customer is on the Pro plan.'),
        ];
    }
}

// Or, without a set at runtime:
$agent->setContext(
    new TextContent("The user is viewing order {$orderId}.")
);
```

Fill the context before the first model call of the turn, because a later change restarts the cache from the question. A structured-output retry adds a correction message, and that attempt carries the context on it.

### Talk to the Agent

We are ready to test how the agent responds to our message based on the new instructions.

```php
use NeuronAI\Chat\Messages\UserMessage;

$message = YouTubeAgent::make()
    ->setThreadId('chat_id')
    ->chat(new UserMessage("Who are you?"))
    ->getMessage();
    
echo $message->getContent();
// Hi, I'm a friendly AI agent specialized in summarizing YouTube videos!
// Can you give me the URL of a YouTube video you want a quick summary of?
```

### Agent State

Since the Agent is an extension of the Workflow, instead of getting the last model response with the `getMessage()` method, you can just run the agent workflow, and get the raw agent state as return value. The agent state contains additional information that can help you inspect what happened during the agent execution.

```php
$state = MyAgent::make()
    ->setThreadId('chat_id')
    ->chat(new UserMessage("Who are you?"));

// $state is an instance of NeuronAI\Agent\AgentState class
$state->getMessage();
```

#### Steps

Calling the `getMessage()` method you are only able to get the last message generated by the model to answer your prompt. But internally the agent can performs many tool call iterations before coming up with the final answer.

The agent state stores the list of all messages between the agent and the provider for the current execution cycle, rather than only the final answer. So you can access the list of messages with the `getSteps()` method on the agent state:

```php
$state = MyAgent::make()
    ->setThreadId('chat_id')
    ->chat(new UserMessage("Who are you?"));

// Access the list of steps during the execution
foreach($state->getSteps() as $message) {
    echo "- ".$message::class."\n";
}

// The final answer
echo $state->getMessage()->getContent();
```

#### Tool Runs

If the agent decide to use tools during the execution, the agent state keeps track the number of tool runs to stop the execution if the [maxRuns](tools.md#max-runs) limit is reached. You can access this map:

```php
$state = MyAgent::make()
    ->setThreadId('chat_id')
    ->chat(new UserMessage("Who are you?"));

// Access the tool runs map
foreach($state->getToolRuns() as $toolName => $runs) {
    echo "- The tool {$toolName} was used {$runs} times\n";
}
```

### Message

The agent always accepts input as a `Message` class, and returns Message instances.

As you saw in the example above we sent a `UserMessage` instance to the agent and we retrieve the reply message that will be an `AssistantMessage` instance. A list of assistant messages and user messages creates a chat.

We will learn more about [ChatHistory](chat-history-and-memory.md) later, but it's important to know that the unified interface for the agent input and output is the `Message` object.

<a href="messages.md" class="button primary" data-icon="arrow-right-long">Learn more about Messages</a>

### Fluent Agent Definition

In alternative to the single class encapsulation you can also instruct the agent inline using the fluent chain of methods:

```php
$agent = Agent::make()
    ->setThreadId('chat_id')
    ->setAiProvider(
        new Anthropic(
            key: 'ANTHROPIC_API_KEY',
            model: 'ANTHROPIC_MODEL',
        )
    )
    ->setInstructions(
        new SystemMessage(...)
    )
    ->addTool([...]);
    
$message = $agent->chat(new UserMessage(...))->getMessage();
echo $message->gentContent();
```

### Extend The Agent Workflow

As you learned above, the Agent class in Neuron is Workflow implementation. The most powerful aspect of the Workflow is that allows you to unpack a process in several steps called `Node`. The core Agent workflow is built with two main nodes the ChatNode, that is responsible to perform the request to the provider, and the ToolNode, responsible to handle Tool execution and approval process.

The Node composition makes you free to extend a node to modify its behaviour, or add other node before and after the core agent loop. To do this the Agent class provides you with two hooks: `entryNodes()`, and `exitNodes()`.

```php
class MyAgent extends Agent
{
  /**
   * @return Node[]
   */
  protected function entryNodes(): array
  {
      return [new AgentStartNode()];
  }

  /**
   * @return Node[]
   */
  protected function exitNodes(): array
  {
      return [new AgentEndNode()];
  }
}
```

With these hooks you can return a list of nodes that will be ran when the `UserMessage` enters into the agent, and after the agent generates the final `AssistantMessage`.

A typical example is the building of a voice agent. In the entry hook you would add a node that process an incoming audio to generate the textual message for the model, and in the exit hook you would transform the text generated by the model into an audio format.

If you want to learn more about the Workflow architecture check out the dedicated section: [Workflow](../workflow/getting-started.md).
