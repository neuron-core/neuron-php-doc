---
description: >-
  Step by step instructions on how to install Neuron in your application and
  create an Agent.
metaLinks:
  alternates:
    - https://app.gitbook.com/s/GHx4l2LknIex7vFIUg1R/overview/getting-started
---

# Installation

### Requirements

* PHP: ^8.1

### Start With One Prompt

Copy & paste this initial prompt to tell your conding agent how to install and configure Neuron AI in your project.

```
You are going to set up Neuron AI, a PHP framework for building agentic applications, 
in this project, and you must follow three steps in order. 

- First: install the framework: check composer.json, and if neuron-core/neuron-ai is not 
already required, run `composer require neuron-core/neuron-ai`;

- Second: install the agent skills that ship 
with the package by running `npx skills add ./vendor/neuron-core/neuron-ai/skills -y` from 
the project root. The skills are symlinked, so they stay current whenever Neuron is 
updated through Composer. From then on, load the relevant skill before you write any Neuron 
code instead of relying on what you remember about the framework, because your memory may 
describe an older version;

- Third: work out what kind of app this is. Look at composer.json 
and the project structure to tell whether it is a Laravel app or a Symfony app, then 
activate the matching skill: neuron-laravel-integration or neuron-symfony-integration. 
If Neuron is already part of the app go through the skill's foundations checklist 
item by item. Report what differs from the recommended setup;

If Neuron is not in the app yet, follow the same checklist to do the first setup. 
If the app uses neither framework, start from a plain PHP agent using 
the neuron-agent skill. Finish with a short summary of what you installed 
and what you found.
```

### Install

Run the command below to install the latest version:

```bash
composer require neuron-core/neuron-ai
```

### Create an Agent

You can easily create your first agent with the Neuron CLI:

{% tabs %}
{% tab title="Unix" %}
```bash
./vendor/bin/neuron make:agent App\\Neuron\\MyAgent
```
{% endtab %}

{% tab title="Windows" %}
```powershell
.\vendor\bin\neuron make:agent App\Neuron\MyAgent
```
{% endtab %}
{% endtabs %}

```php
namespace App\Neuron;

use NeuronAI\Agent\Agent;
use NeuronAI\Agent\SystemPrompt;
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

$message = MyAgent::make()
    ->setThreadId('chat_id')
    ->chat(new UserMessage("Hi, who are you?"))
    ->getMessage();

echo $message->getContent();
// I'm a friendly AI Agent built with Neuron, how can I help you today?
```

### Monitoring & Debugging

Many of the applications you build with Neuron will contain multiple steps with multiple invocations of LLM calls. As these applications get more and more complex, it becomes crucial to be able to inspect what exactly is going on inside your agentic system. The best way to do this is with [Inspector](https://inspector.dev/).

{% embed url="https://docs.inspector.dev/guides/neuron-ai" %}

### Video Tutorial

{% embed url="https://www.youtube.com/watch?v=oSA1bP_j41w" %}
