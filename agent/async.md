---
description: Run workflows in an async context.
---

# Async

Neuron supports asynchronous execution and parallel processing of agents, enabling you to efficiently handle multiple operations simultaneously. Neuron's approach to asynchronous execution offers several advantages:

**Framework Agnostic**: You can make agents and RAG async friendly for the most common async environments with a simple adapter, no changes are required to your implementation.

**Providers Compatibility**: Our solution works regardless of the provider you use. You can have workflow or multi-agent systems using different providers and models running smoothly in an async loop.

**Scalability**: Applications handling large volumes of data (products classification, content moderation, data labeling) benefit significantly from concurrent processing capabilities.

### Concurrency vs Async

Concurrency is the high-level concept of managing tasks, which can involve multiple threads/cores (parallelism), whereas async uses event loops/callbacks to let tasks run out of order, perfect for I/O-bound work without waiting.&#x20;

Concurrency is about managing many things in parallel, while asynchrony is how a single thread can manage many I/O operations efficiently.

AI Agents are typically considered I/O heavy software because the HTTP request to run inference on the model usually takes seconds to complete. In a standard PHP enviornment during this time your application just wait. In this section of the documentation we provide you with a couple of solutions to run multiple agent efficiently.

## Concurrency

You don't need any particular feature from Neuron to run multiple agents in parallel. You only need your PHP application to be able to span multiple processes to handle the execution of multiple agents at the same time. You can do this with PHP libraries like [spatie/fork](https://github.com/spatie/fork), or framework specific solutions like [Laravel concurrency](https://laravel.com/docs/master/concurrency), or [Symfony process](https://symfony.com/doc/current/components/process.html).&#x20;

Async is a different story.&#x20;

## Async

Previous versions of Neuron were strongly coupled with the Guzzle client to perform HTTP requests for model inference on the providers API. Guzzle is a great tool, but it's not compatible with truly async event loops like those provided by frameworks like [Amp](https://github.com/amphp/amp) and [ReactPHP](https://github.com/reactphp/reactphp).

In order to run agents in such async environments it's required to integrate with their specific implementations. That's why Neuron ships with a simple `HttpClientInterface` that can be implemented to allow AI providers run HTTP requests smoothly in an async loop.&#x20;

By default the framework uses the Guzzle implementation, but you can inject custom HTTP clients based on your needs. We already provide implementations for the most common async framework.

### AmpHttpClient

If you want to use Amp to run multiple async agent requests you need to install  `amphp/http-client` .

```bash
composer require amphp/http-client
```

Now you can inject the built-in `AmpHttpClient` adapter into the provider:

```php
namespace App\Neuron;

use NeuronAI\Agent\Agent;
use NeuronAI\HttpClient\AmpHttpClient;
use NeuronAI\Providers\Anthropic\Anthropic;

class MyAgent extends Agent
{
    protected function provider(): AIProviderInterface
    {
        // It's the same for any provider (Anthropic, OpenAI, Ollama, Gemini, etc.)
        return new Anthropic(
            key: 'ANTHROPIC_API_KEY',
            model: 'ANTHROPIC_MODEL',
            httpClient: new AmpHttpClient(),
        );
    }
}
```

Now you can run multiple agent requests using Amp async/await pattern to run multiple agent asynchronously:

```php
use Amp\Future;

use function Amp\async;

$handler1 = MyAgent::make()->chat(new UserMessage('Hi!'));
$handler2 = MyAgent::make()->chat(new UserMessage('Hi!'));
$handler3 = MyAgent::make()->chat(new UserMessage('Hi!'));

// Run three requests in parallel
[$response1, $response2, $response3] = Future\await([
    async(fn() => $handler1->getMessage()), 
    async(fn() => $handler2->getMessage()),
    async(fn() => $handler3->getMessage()),
]);

// Print the content
echo $response1->getContent();
echo $response2->getContent();
echo $response3->getContent();
```

### Async RAG

The HTTP client abstraction is also accepted by all the other framework components, like embedding providers and vector stores. You can provide an async client to all this components and also run data loading pipeline asynchronously.

```php
class MyChatBot extends RAG
{
    protected function provider(): AIProviderInterface
    {
        return new Anthropic(
            key: 'ANTHROPIC_API_KEY',
            model: 'ANTHROPIC_MODEL',
            httpClient: new AmpHttpClient(),
        );
    }
    
    protected function embeddings(): EmbeddingsProviderInterface
    {
        return new VoyageEmbeddingsProvider(
            key: 'OPENAI_API_KEY',
            model: 'OPENAI_MODEL'
            httpClient: new AmpHttpClient(),
        );
    }
    
    protected function vectorStore(): VectorStoreInterface
    {
        return new PineconeVectorStore(
            key: 'PINECONE_API_KEY',
            indexUrl: 'PINECONE_INDEX_URL',
            httpClient: new AmpHttpClient(),
        );
    }
}
```

### Guardrails

The ability to execute agents asynchonously opens the possibility of running an input guardrail at the same time as the main request. Typically you can create a specialized agent to run security checks against the user input, and use the structured reponse to retrieve the result.

You can run both requests in parallel, and check the guardrail result before returning the response to the user. Running both requests in parallel allows you to enforce security without impacting the user experience.

```php
use Amp\Future;
use function Amp\async;

$input = new UserMessage('Hi!');

// Run three requests in parallel
[$response, $guardrail] = Future\await([
    async(fn() => MyAgent::make()->chat($input)->getMessage()), 
    async(fn() => GuardrailAgent::make()->structured($input, Guardrail::class)),
]);

if (! $guardrail->valid) {
    throw \Exception('Content policy violation.');
}

// Print the content
echo $response->getContent();
```
