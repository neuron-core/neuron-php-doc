---
description: >-
  Step by step instructions on how to install Neuron in your application and
  create an Agent.
---

# Installation

{% hint style="warning" %}
### Neuron v3 scheduled for the end of February.

Start with it or read the upgrade guide to update your project.&#x20;

[**Upgrade guide**](https://docs.neuron-ai.dev/neuron-v3/overview/upgrade)
{% endhint %}

### Requirements

* PHP: ^8.1

### Install

Run the composer command below to install the latest version:

```bash
composer require neuron-core/neuron-ai
```

### Create an Agent

You can easily create your first agent with command below:

{% tabs %}
{% tab title="Unix" %}
```bash
php vendor/bin/neuron make:agent App\\Neuron\\MyAgent
```
{% endtab %}

{% tab title="Windows" %}
```powershell
php .\vendor\bin\neuron make:agent App\Neuron\MyAgent
```
{% endtab %}
{% endtabs %}

```php
namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\SystemPrompt;
use NeuronAI\Providers\Anthropic\Anthropic;

class MyAgent extends Agent
{
    protected function provider(): AIProviderInterface
    {
        // return an AI provider (Anthropic, OpenAI, Ollama, Gemini, etc.)
        return new Anthropic(
            key: 'ANTHROPIC_API_KEY',
            model: 'ANTHROPIC_MODEL',
        );
    }

    public function instructions(): string
    {
        return (string) new SystemPrompt(
            background: ["You are a friendly AI Agent created with Neuron framework."],
        );
    }
}
```

### Talk to the Agent

Send a prompt to the agent to get a response from the underlying LLM:

```php
use NeuronAI\Chat\Messages\UserMessage;

$response = MyAgent::make()->chat(
    new UserMessage("Hi, Who are you?")
);
    
echo $response->getContent();

// I'm a friendly AI Agent built with Neuron, how can I help you today?
```



### Monitoring & Debugging

Many of the applications you build with Neuron will contain multiple steps with multiple invocations of LLM calls, tools, external memory system, etc. As these applications get more and more complex, it becomes crucial to be able to inspect what exactly is going on inside your agentic system. The best way to do this is with [Inspector](https://inspector.dev/).

After you sign up at the link above, make sure to set the `INSPECTOR_INGESTION_KEY` variable in the application environment file to start monitoring:

{% code title=".env" %}
```
INSPECTOR_INGESTION_KEY=nwse877auxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```
{% endcode %}

### Video Tutorial On A Laravel Application

{% embed url="https://www.youtube.com/watch?v=oSA1bP_j41w" %}
