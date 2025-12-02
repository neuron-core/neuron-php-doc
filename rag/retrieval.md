---
description: Implement custom retrieval strategies
---

# Retrieval

### Introduction

The RAG module has a separate retrieval component that allows you to implement different strategies to accomplish context retrieval from an external data source. By default RAG uses `SimilarityRetrieval` that simply query the vector store to retrieve documents:

```php
namespace App\Neuron;

use NeuronAI\Providers\AIProviderInterface;
use NeuronAI\Providers\Anthropic\Anthropic;
use NeuronAI\RAG\Embeddings\EmbeddingsProviderInterface;
use NeuronAI\RAG\Embeddings\OpenAIEmbeddingsProvider;
use NeuronAI\RAG\RAG;
use NeuronAI\RAG\VectorStore\FileVectorStore;
use NeuronAI\RAG\VectorStore\VectorStoreInterface;
use NeuronAI\Tools\Toolkits\Calculator\CalculatorToolkit;

class WorkoutTipsAgent extends RAG
{
    protected function retrieval(): RetrievalInterface
    {
        return new SimilarityRetrieval(
            $this->vectorStore(),
            $this->embeddings()
        );
    }
    
    protected function embeddings(): EmbeddingsProviderInterface
    {
        return new OpenAIEmbeddingsProvider(
            key: 'OPENAI_API_KEY',
            model: 'OPENAI_MODEL'
        );
    }
    
    protected function vectorStore(): VectorStoreInterface
    {
        return new FileVectorStore(
            directory: __DIR__,
            name: 'demo'
        );
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
