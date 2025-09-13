---
description: Learn what Neuron is and what you can do with it.
---

# Introduction

{% hint style="warning" %}
### 🚨 IMPORTANT: Repository Migration Notice

**Effective October 1st, 2025**, the official Neuron repository will be migrating from the Inspector GitHub organization to a dedicated [**Neuron organization**](https://github.com/neuron-core).

For detailed migration instructions and configuration updates, please visit our [Repository Migration Guide](https://docs.neuron-ai.dev/overview/readme/repository-migration).
{% endhint %}

### What is Neuron

Neuron is a PHP framework for creating and orchestrating AI Agents. It allows you to integrate AI entities in your existing&#x20;PHP applications with a powerful and flexible architecture.&#x20;

We provide tools for the entire agentic application development lifecycle, from LLM interfaces, to data loading, to multi-agent orchestration, to monitoring and debugging. In addition, we provide [tutorials and other educational content](overview/fast-learning-by-video.md) to help you get started using AI Agents in your projects.

<figure><img src=".gitbook/assets/neuron-ai-architecture.png" alt=""><figcaption><p>Neuron architecture</p></figcaption></figure>

### Developer Experience

Neuron's architecture prioritizes the fundamentals that experienced engineers expect from production-grade software.&#x20;

***

#### Strong Typing System

The framework leverages PHP 8's mature type system throughout its codebase, with every method signature, property, and return value explicitly typed. The entire framework passes PHPStan 100% type coverage.

***

#### IDE Friendly

The strongly-typed approach means your IDE can provide accurate autocompletion for agent configurations, tool parameters, and response handling. Method signatures include detailed PHPDoc annotations that provide context beyond type hints when needed, explaining parameter expectations and return value structures.

***

This foundation allows faster debugging cycles, easy integration patterns with frameworks like Symfony or Laravel. We assume you're building systems that need to be maintained, extended, and understood by teams rather than individual experiments.

### Carefully Crafted Architecture

Whether you're working within a Laravel application, a Symfony project, a WordPress plugin, or a custom MVC framework, Neuron integrates seamlessly with your existing codebase without refactoring or disrupting established environments.

Neuron uses standard PSR interfaces where appropriate and maintains minimal external dependencies, avoiding conflits across different PHP environments and framework versions. This design choice prevents the common problem where introducing a new library increase the risks of getting stuck due to incompatible versions of dependencies.

For teams working across multiple projects, this approach provides consistency. The same Neuron patterns and implementations work regardless of whether you're building a new microservice in pure PHP, extending a WordPress site, or adding features to an enterprise Symfony and Laravel application. Knowledge transfer between projects becomes seamless, and developers can leverage their Neuron expertise across their entire PHP portfolio.

### Community Driven

These design principles create a unified ecosystem for AI development across all PHP communities. Rather than fragmenting innovation across framework-specific solutions, Neuron enables collaboration between Laravel developers, Symfony contributors, WordPress plugin authors, and custom framework maintainers. When improvements are made to Neuron's core capabilities, they benefit every PHP developer regardless of their architectural preferences.

The result is a larger, more diverse community working toward common goals. Framework-specific AI libraries naturally limit their contributor base to developers familiar with that particular framework. Neuron's universal approach attracts contributors from across the PHP ecosystem, leading to more robust implementations, broader testing across different environments, and faster development of new features. This collaborative approach also means better support for newcomers, as experienced developers from various PHP backgrounds can provide guidance and assistance regardless of which framework someone happens to be learning alongside Neuron.

### Production Readiness

Integrating AI Agents into your application you’re not working only with functions and deterministic code, you program your agent also influencing probability distributions. Same input ≠ output. That means reproducibility, versioning, and debugging become real problems.

The [Inspector](https://inspector.dev/) team designed Neuron with built-in observability features, so you can monitor AI agents were running, helping you maintain production-grade implementations with confidence.

Error handling and retry mechanisms are built into the framework, ensuring your agents can gracefully handle failures, rate limits, and other common issues in production environments.&#x20;

### Support For Multiple Providers

Neuron uses a common interface for large language models (`AIProviderInterface`) as well as for the other components, such as [embedding](components/embeddings-provider.md), [vector stores](components/vector-store.md), [toolkits](getting-started/tools.md#toolkits-composable-agent-capabilities), etc. The modular architecture allows you to swap components as needed, whether you're changing language model providers, adjusting memory backends, or scaling across multiple servers.

{% tabs %}
{% tab title="Anthropic" %}
```php
namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\Chat\Messages\UserMessage;
use NeuronAI\Providers\AIProviderInterface;
use NeuronAI\Providers\Anthropic\Anthropic;

class MyAgent extends Agent
{
    protected function provider(): AIProviderInterface
    {
        return new Anthropic(
            key: 'ANTHROPIC_API_KEY',
            model: 'ANTHROPIC_MODEL',
        );
    }
}

echo MyAgent::make()->chat(new UserMessage("Hi!"));
// Hi, how can I help you today?
```
{% endtab %}

{% tab title="Ollama" %}
```php
namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\Chat\Messages\UserMessage;
use NeuronAI\Providers\AIProviderInterface;
use NeuronAI\Providers\Ollama\Ollama;

class MyAgent extends Agent
{
    protected function provider(): AIProviderInterface
    {
        return new Ollama(
            url: 'OLLAMA_URL',
            model: 'OLLAMA_MODEL',
        );
    }
}

echo MyAgent::make()->chat(new UserMessage("Hi!"));
// Hi, how can I help you today?
```
{% endtab %}

{% tab title="OpenAI" %}
```php
namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\Chat\Messages\UserMessage;
use NeuronAI\Providers\AIProviderInterface;
use NeuronAI\Providers\OpenAI\OpenAI;

class MyAgent extends Agent
{
    protected function provider(): AIProviderInterface
    {
        return new OpenAI(
            key: 'OPENAI_API_KEY',
            model: 'OPENAI_MODEL',
        );
    }
}

echo MyAgent::make()->chat(new UserMessage("Hi!"));
// Hi, how can I help you today?
```
{% endtab %}

{% tab title="Gemini" %}
```php
namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\Chat\Messages\UserMessage;
use NeuronAI\Providers\AIProviderInterface;
use NeuronAI\Providers\Gemini\Gemini;

class MyAgent extends Agent
{
    protected function provider(): AIProviderInterface
    {
        return new Gemini(
            key: 'GEMINI_API_KEY',
            model: 'GEMINI_MODEL',
        );
    }
}

echo MyAgent::make()->chat(new UserMessage("Hi!"));
// Hi, how can I help you today?
```
{% endtab %}
{% endtabs %}

Check out all the supported providers in the [AI Provider](components/ai-provider.md) section.

## What is an AI Agent

An AI agent is a software component whose output is generated by an Artificial Intelligence. These components can understand and respond to customer inquiries without human intervention. They are created using an agent development kit, like Neuron, to connect additional components and handle a wide range of tasks. These intelligent agents can include anything from answering simple questions to resolving complex issues that require reasoning, decision making, and proactive interactions with external systems.

Compared to a raw LLM (that primarily provide information and respond to questions within a conversation), AI agents can augment their knowledge with external sources, and take independent actions to complete tasks.

While a simple LLM can answer your questions directly during a conversation, an AI agent might be able to:

* Research information across multiple websites and compile it for you
* Manage your email by responding to simple messages
* Read data from your database and alert you via email when something important happens

The key characteristic that distinguishes agents from traditional software is their ability to operate with incomplete information and adapt to changing requirements.

### Why Build AI Agents in PHP?&#x20;

PHP remains one of the most widely deployed server-side languages, powering the majority of web applications worldwide. If you're already working with PHP, Neuron allows you to integrate AI capabilities directly into your existing codebase without learning new languages or restructuring your applications. This approach significantly reduces the barrier to entry for adding intelligence to web app, content management systems, e-commerce platforms, and business backend applications.

Modern PHP offers robust object-oriented programming features, strong typing capabilities, and excellent performance characteristics that make it well-suited for AI Agents development. The language's mature ecosystem, and straightforward deployment model provide a solid foundation for building reliable agentic systems.

### Getting Started With A Video Tutorial

{% embed url="https://www.youtube.com/watch?v=oSA1bP_j41w" %}

## The Multi-Agent Problem&#x20;

Building AI applications quickly becomes complex when multiple agents need to collaborate. Managing state between agents, handling failures gracefully,  and maintaining conversation context across different AI services creates intricate dependency chains. Without proper orchestration, developers often resort to brittle conditional logic, manual state management, and sequential processing that doesn't scale. The challenge isn't just technical—it's architectural. How do you design systems where agents can work together, share information, and recover from failures while maintaining clear, debuggable code?

<a href="broken-reference" class="button secondary" data-icon="arrow-progress">Neuron Workflow</a>

## Ecosystem

### [E-Book - "Start With AI Agents In PHP"](https://www.amazon.it/dp/B0F1YX8KJB)

The gap between modern agentic technologies and traditional PHP development has been widening in recent years. While Python developers enjoy a wealth of libraries and frameworks to create AI Agents, PHP developers have often been left wondering how they can participate in this technological revolution without completely retooling their skillsets or rebuilding their applications from scratch.

Neuron changes all that.

This book serves as both an introduction to AI Agents concepts for developers and a comprehensive guide to Neuron framework.

<a href="https://www.amazon.com/dp/B0F1YX8KJB" class="button secondary" data-icon="amazon">Get on Amazon</a>&#x20;

<a href="https://play.google.com/store/books/details?pcampaignid=books_read_action&#x26;id=agJPEQAAQBAJ&#x26;pli=1" class="button secondary" data-icon="google">Get on GooglePlay</a>

### [Newsletter](https://neuron-ai.dev)

Register to the Neuron internal [newsletter](https://neuron-ai.dev/) to get informative papers, articles, and best practices on how to start with AI development in PHP.

You will learn how to approach AI systems in the right way, understand the most important technical concepts behind LLMs, and how to start implementing your AI solutions into your PHP application with the Neuron AI framework.

### [Forum](https://github.com/inspector-apm/neuron-ai/discussions)

We’re using [Discussions](https://github.com/inspector-apm/neuron-ai/discussions) as a place to connect with PHP developers working on Neuron to create their Agentic applications. We hope that you:

* Ask questions you’re wondering about.
* Share ideas.
* Engage with other community members.
* Welcome others and are open-minded.

### [**Inspector.dev**](https://inspector.dev)

Neuron is part of the Inspector ecosystem as a trustable platform to create reliable and scalable AI driven solutions.&#x20;

Trace and evaluate your agents execution flow to help you maintain production grade implementations with confidence. Check out the [**monitoring integrations**](getting-started/observability.md).

## Core components

* [**AI Provider**](components/ai-provider.md)
* [**Toolkit**](getting-started/tools.md#toolkits-composable-agent-capabilities)
* [**Embeddings Provider**](components/embeddings-provider.md)
* [**Data Loader**](components/data-loader.md)
* [**Vector Store**](components/vector-store.md)
* [**Chat History**](components/chat-history-and-memory.md)
* [**MCP connector**](getting-started/mcp-connector.md)
* [**Monitoring & Debugging**](getting-started/observability.md)
* [**Pre/Post Processors**](components/pre-post-processor.md)
* [**Workflow**](broken-reference)

## Additional Resources

* Repository: [https://github.com/inspector-apm/neuron-ai](https://github.com/inspector-apm/neuron-ai)
* Inspector: [https://inspector.dev](https://inspector.dev)
* E-Book: [https://www.amazon.it/dp/B0F1YX8KJB](https://www.amazon.it/dp/B0F1YX8KJB)
