---
description: Implement custom retrieval strategies
metaLinks:
  alternates:
    - https://app.gitbook.com/s/GHx4l2LknIex7vFIUg1R/rag/retrieval
---

# Retrieval

### Introduction

The RAG module has a separate retrieval component that allows you to implement different strategies to accomplish context retrieval from external data sources. By default, RAG uses `SimilarityRetrieval` that simply queries the vector store to retrieve documents:

```php
namespace App\Neuron;

use NeuronAI\Providers\AIProviderInterface;
use NeuronAI\RAG\Embeddings\EmbeddingsProviderInterface;
use NeuronAI\RAG\RAG;
use NeuronAI\RAG\RAG\Retrieval\RetrievalInterface;
use NeuronAI\RAG\RAG\Retrieval\SimilarityRetrieval;
use NeuronAI\RAG\VectorStore\VectorStoreInterface;

class WorkoutTipsAgent extends RAG
{
    protected function retrieval(): RetrievalInterface
    {
        return new SimilarityRetrieval(
            $this->resolveVectorStore(),
            $this->resolveEmbeddingsProvider()
        );
    }
    
    protected function provider(): AIProviderInterface
    {
        // Return an instance of an AI provider...
    }
    
    protected function embeddings(): EmbeddingsProviderInterface
    {
        // Return an embeddings provider instance...
    }
    
    protected function vectorStore(): VectorStoreInterface
    {
        // Return a vector store instance...
    }
}
```

Implementing `RetrievalInterface` you are free to create any custom retrieval behaviour for your RAG.

```php
interface RetrievalInterface
{
    /**
     * Retrieve relevant documents for the given query.
     *
     * @return Document[]
     */
    public function retrieve(Message $query): array;
}
```

If you are implementing custom workflow you can use retrieval as a standalone component to dynamically retrieve context data for use in your agentic systems.

### Filters

```php
use NeuronAI\RAG\Retrieval\SimilarityRetrieval;
use NeuronAI\RAG\VectorStore\Filter\Filter;
use NeuronAI\RAG\VectorStore\Filter\FilterGroup;

class WorkoutTipsAgent extends RAG
{
    protected function retrieval(): RetrievalInterface
    {
        return new SimilarityRetrieval(
            $this->resolveVectorStore(),
            $this->resolveEmbeddingsProvider(),
            filters: FilterGroup::and(Filter::eq('tenant', $tenantId)),  // optional
        );
    }
}
```

<a href="rag.md#filters" class="button primary" data-icon="arrow-right-long">More about Filters</a>

### Retrieval as a Tool

Neuron provides you with a built-in `RetrievalTool` tool that enables AI agents to perform context retrieval from vector stores if the model think it needs more context to answer the current user question. It's built on top of the `RetrievalInterface` , making it possible to build agents with on-demand RAG (Retrieval Augmented Generation)&#x20;capabilities instead of the authomatic context injection provided by the RAG component.

Here is an example using the built-in `SimilarityRetrieval`:

```php
use NeuronAI\Tools\Toolkits\RetrievalTool;
use NeuronAI\RAG\Retrieval\SimilarityRetrieval;

class AgenticRAG extends Agent
{
    protected function provider(): AIProviderInterface
    {...}
    
    protected function instructions(): string
    {...}
    
    protected function tools(): array
    {
        return [
            RetrievalTool::make(
                new SimilarityRetrieval(
                    $this->vectorStore(), 
                    $this->embeddings()
                )
            ),
        ];
    }
    
    protected function vectorStore(): VectorStoreInterface
    {
        return new FileVectorStore(__DIR__);
    }
    
    protected function embeddings(): EmbeddingsProviderInterface
    {
        return new OllamaEmbeddingsProvider(
            model: 'OLLAMA_EMBEDDINGS_MODEL'
        );
    }
}
```

As you can notice in this example we don't extend RAG but the basic Agent instead. In this implementation we let the model decide if it's the case to search an external source to answer the user question.

You can always use all the tool and agent methods to customize description, instructions, and prompts in general to make the model behave according to your use case.

### Semantic Memory

Using RAG you can activate the retrieval of memory across threads. Users with multiple active threads can see the agent remember their past conversations and preferences.

```php
use NeuronAI\RAG\Retrieval\SemanticMemoryRetrieval;

class MyAgent extends RAG
{
    // Define the usual provider(), vectorStore(), and embeddings() hooks.
    
    protected function retrieval(): RetrievalInterface
    {
        return new SemanticMemoryRetrieval(
            vectorStore: $this->resolveVectorStore(),
            embeddingProvider: $this->resolveEmbeddingProvider(),
            threadIds: $threadIds,
        );
    }
}
```

The argument `threadIds` is the list of threads you want to recall memory from. It can be the full list of user active threads or a subset of them based on your application preferences.

{% hint style="warning" %}
Remeber to exclude the current **`threadId`** from the recall list, otherwise the messages on the same conversation will be duplicated into the system instructions of the agent.
{% endhint %}

To activate memory creation for the current conversation you need to attach the `ConversationIngestionNode` as the exit node, so the RAG can automatically store messages in the assigned vector store.

```php
use NeuronAI\RAG\Nodes\ConversationIngestionNode;

class MyAgent extends RAG
{
    // Define the usual provider(), vectorStore(), and embeddings() hooks.
    
    protected function retrieval(): RetrievalInterface
    {
        return new SemanticMemoryRetrieval(
            vectorStore: $this->resolveVectorStore(),
            embeddingProvider: $this->resolveEmbeddingProvider(),
            threadIds: $threadIds,
        );
    }
    
    protected function exitNodes(): array
    {
        return [
            // Store the last user/assistant pair in the memroy store
            new ConversationIngestionNode(
                vectorStore: $this->resolveVectorStore(),
                embeddingProvider: $this->resolveEmbeddingsProvider(),
                chatHistory: $this->getChatHistory(),
            )
        ];
    }
}
```

### Composite Retrieval

`CompositeRetrieval` allows you to concatenate multiple retrieval task in a pipeline. It calls its children in order and combines their results. Each child receives the same preprocessed query and mandatory per-run filters.&#x20;

Put collection-specific filters on that child rather than in the shared retrieval scope. Shared filters must be supported by every child and are never dropped. RAG then deduplicates by content and runs the common postprocessors.

```php
use NeuronAI\RAG\Retrieval\CompositeRetrieval;

class MyAgent extends RAG
{
    // Define the usual provider(), vectorStore(), and embeddings() hooks.
    
    protected function retrieval(): RetrievalInterface
    {
        return new CompositeRetrieval([
            new SemanticMemoryRetrieval(
                vectorStore: $this->resolveVectorStore(),
                embeddingProvider: $this->resolveEmbeddingsProvider(),
                threadIds: $authorizedThreadIds,
            ),
            new SimilarityRetrieval(
                vectorStore: $this->resolveVectorStore(),
                embeddingProvider: $this->resolveEmbeddingsProvider(),
            ),
        ]);
    }
}
```

The code above uses the same vector store for both retrieval strategies, but you would prorbably have two separate stores. You are free to combine components to reflect your application needs and architecture.

### RAPTOR Retrieval

Most retrieval-augmented models work by breaking down documents into small chunks and retrieving only the most relevant ones. However, this approach has some limitations:

* **Loss of Context**: Retrieving only small, isolated chunks may miss the bigger picture especially for documents with long contexts.
* **Difficulty in Multi-Step Reasoning**: Some questions require information from multiple sections of a document.

**Use RAPTOR when:**

* Users ask open-ended questions that require comprehensive coverage
* Your domain involves complex topics where context matters as much as facts
* You need to handle queries about themes, trends, or relationships across documents

**Stick with traditional RAG when:**

* Users primarily need quick, specific fact retrieval
* Processing speed and token efficiency are critical constraints

Learn more about RAPTOR in the dedicated repository:

{% embed url="https://github.com/neuron-core/raptor-retrieval" %}
