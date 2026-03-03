---
description: Implement custom retrieval strategies
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

### RAPTOR Retrieval Module

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
