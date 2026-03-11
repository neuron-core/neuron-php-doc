---
description: Easily implement LLM interactions with built-in memory and tool usage.
---

# Agent

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
        return (string) new SystemPrompt(
            background: ["You are a friendly AI Agent created with Neuron framework."],
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

After you sign up at the link above, make sure to set the `INSPECTOR_INGESTION_KEY` variable in the application environment file to start monitoring:

{% code title=".env" %}
```
INSPECTOR_INGESTION_KEY=nwse877auxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```
{% endcode %}

### AI Provider

The minimum implementation requires assigning an AI Provider that will be the language and reasoning engine of your agent.

The only required method to implement is `provider()`  returning the instance of the provider you want to use. Let's assume it's Anthropic.

```php
<?php

namespace App\Neuron;

use NeuronAI\Agent\Agent;
use NeuronAI\Agent\SystemPrompt;
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
    
    protected function instructions(): string
    {
        return (string) new SystemPrompt(
            background: ["You are a friendly AI Agent created with Neuron framework."],
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
use NeuronAI\Agent\SystemPrompt;
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
    
    protected function instructions(): string
    {
        return (string) new SystemPrompt(
            background: ["You are an AI Agent specialized in writing YouTube video summaries."],
            steps: [
                "Get the url of a YouTube video, or ask the user to provide one.",
                "Use the tools you have available to retrieve the transcription of the video.",
                "Write the summary.",
            ],
            output: [
                "Write a summary in a paragraph without using lists. Use just fluent text.",
                "After the summary add a list of three sentences as the three most important take away from the video.",
            ]
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

The `SystemPrompt` class is designed to take your base instructions and build a consistent prompt for the underlying model reducing the effort for prompt engineering. The properties has the following meaning:

* **background**: Write about the role of the Agent. Think about the macro tasks it's intended to accomplish.
* **steps**: Define the way you expect the Agent to behave. Multiple steps help the Agent to act consistently.
* **output**: Define how you want the agent to respond. Be explicit on the format you expect.

We highly recommend to use the `SystemPrompt` class to increase the quality of the results, in alternative you can just return a simple string:

```php
<?php

namespace App\Neuron;

use NeuronAI\Agent\Agent;

class YouTubeAgent extends Agent
{
    ...
    
    protected function instructions(): string
    {
        return "You are an AI Agent specialized in writing YouTube video summaries.";
    }
}
```

### Talk to the Agent

We are ready to test how the agent responds to our message based on the new instructions.

```php
use NeuronAI\Chat\Messages\UserMessage;

$message = YouTubeAgent::make()
    ->chat(new UserMessage("Who are you?"))
    ->getMessage();
    
echo $message->getContent();
// Hi, I'm a frindly AI agent specialized in summarizing YouTube videos!
// Can you give me the URL of a YouTube video you want a quick summary of?
```

### Agent State

Since the Agent is an extension of the Workflow, instead of getting the last model response with the `getMessage()` method, you cvan just run the agent workflow, and get the raw agent state as return value. The agent state contains additional information that can help you inspect what happened during the agent execution.

```php
$state = MyAgent::make()
    ->chat(new UserMessage("Who are you?"))
    ->run();

// $state is an instance of NeuropnAI\Agent\AgentState class
$state->getMessage();
```

#### Steps

Calling the `getMessage()` method you are only able to get the last message generated by the model to answer your prompt. But internally the agent can performs many tool call iterations before coming up with the final answer.

The agent state stores the list of all messages between the agent and the provider for the current execution cycle, rather than only the final answer. So you can access the list of messages with the `getSteps()` method on the agent state:

```php
$state = MyAgent::make()
    ->chat(new UserMessage("Who are you?"))
    ->run();

// Access the list of steps during the execution
foreach($state->getSteps() as $message) {
    echo "- ".$message::class."\n";
}

// The final answer
echo $state->getMessage()->getContent();
```

#### Tool Runs

If the agent decide to use tools during the execution, the agent state keeps track iof thethe number of tool runs to stop the execution if the [maxRuns](tools.md#max-runs) limit is reached. You can access this map:

```php
$state = MyAgent::make()
    ->chat(new UserMessage("Who are you?"))
    ->run();

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
    ->setAiProvider(
        new Anthropic(
            key: 'ANTHROPIC_API_KEY',
            model: 'ANTHROPIC_MODEL',
        )
    )
    ->setInstructions(
        (string) new SystemPrompt(...)
    )
    ->addTool([...]);
    
$message = $agent->chat(new UserMessage(...))->getMessage();
echo $message->gentContent();
```
