---
description: >-
  Neuron provides you with ready to use components to connect your agent to
  vector databases.
metaLinks:
  alternates:
    - https://app.gitbook.com/s/GHx4l2LknIex7vFIUg1R/rag/vector-store
---

# Vector Store

We currently offer first-party support for the following vector store:

### Memory

This is an implementation of a volatile vector store that keeps your embeddings into the machine memory for the current session. It's useful when you don't need to store the generated embeddings for long term use, but just during current interaction sessions (or for local use).

```php
namespace App\Neuron;

use NeuronAI\RAG\RAG;
use NeuronAI\RAG\VectorStore\MemoryVectorStore;
use NeuronAI\RAG\VectorStore\VectorStoreInterface;

class MyChatBot extends RAG
{
    ...

    protected function vectorStore(): VectorStoreInterface
    {
        return new MemoryVectorStore();
    }
}
```

### File

File storage could be useful for low volume use case or local and staging environments. Embedded documents will be stored in the file system and processed during similarity search.

`FileVectorStore` uses PHP generators to read the embedded documents from the file systems. It will never keep more than `topK` items in memory while iterating very fast. You can store thousands of documents in your local filesystem only taking care on the maximum time you can accept to perform the similarity search.

You can also use this component to release agents with some knowledge already incorporated in a file.

```php
namespace App\Neuron;

use NeuronAI\RAG\RAG;
use NeuronAI\RAG\VectorStore\FileVectorStore;
use NeuronAI\RAG\VectorStore\VectorStoreInterface;

class MyChatBot extends RAG
{
    ...

    protected function vectorStore(): VectorStoreInterface
    {
        return new FileVectorStore(
            directory: storage_path(),
            topK: 4
        );
    }
}
```

### PHPVector

PHPVector adapter on top of [`ezimuel/phpvector`](https://github.com/ezimuel/PHPVector). It is a pure-PHP vector database implementing **HNSW** (Hierarchical Navigable Small World) for approximate nearest-neighbour search and **BM25** for full-text retrieval. Both engines can be combined into a single **hybrid search** pipeline.

You can install the component with composer:

```shellscript
composer require ezimuel/phpvector
```

Use it in a RAG context:

```php
namespace App\Neuron;

use NeuronAI\RAG\RAG;
use NeuronAI\PHPVector\PHPVector;
use NeuronAI\RAG\VectorStore\VectorStoreInterface;

class MyRAG extends RAG
{
    ...

    protected function vectorStore(): VectorStoreInterface
    {
        return new PHPVector(
            path: '/var/data/mydb',
            topK: 5,
        );
    }
}
```

### MariaDB

MariaDB supports VECTOR column type starting from version 11.7. To make this component works you need to create the table to store documents and related vectors. Here is the SQL script you can use to do so:

```sql
CREATE TABLE IF NOT EXISTS rag_documents (
    id UUID NOT NULL PRIMARY KEY,
    content TEXT,
    sourceType VARCHAR(255),
    sourceName VARCHAR(255),
    metadata JSON,
    embedding VECTOR(1536) NOT NULL,
    VECTOR INDEX (embedding)
)
```

Here is how to use the component in your RAG:

```php
namespace App\Neuron;

use NeuronAI\RAG\RAG;
use NeuronAI\RAG\VectorStore\MariaDBVectorStore;
use NeuronAI\RAG\VectorStore\VectorStoreInterface;

class MyChatBot extends RAG
{
    ...

    protected function vectorStore(): VectorStoreInterface
    {
        return new MariaDBVectorStore(
            new \PDO(...), // Or get the PDO instance from the ORM
        );
    }
}
```

### Pinecone

Pinecone makes it easy to provide long-term memory for high-performance AI applications. It’s a managed, cloud-native vector database with a simple API and no infrastructure hassles. Pinecone serves fresh, filtered query results with low latency at the scale of billions of vectors.

Here is how to use Pinecone in your agent:

```php
namespace App\Neuron;

use NeuronAI\RAG\RAG;
use NeuronAI\RAG\VectorStore\PineconeVectorStore;
use NeuronAI\RAG\VectorStore\VectorStoreInterface;

class MyChatBot extends RAG
{
    ...

    protected function vectorStore(): VectorStoreInterface
    {
        return new PineconeVectorStore(
            key: 'PINECONE_API_KEY',
            indexUrl: 'PINECONE_INDEX_URL'
        );
    }
}
```

### Weaviate

```php
namespace App\Neuron;

use NeuronAI\RAG\RAG;
use NeuronAI\RAG\VectorStore\WeaviateVectorStore;
use NeuronAI\RAG\VectorStore\VectorStoreInterface;

class MyChatBot extends RAG
{
    ...

    protected function vectorStore(): VectorStoreInterface
    {
        return new WeaviateVectorStore(
            collection: 'WEAVIATE_COLLECTION_NAME',
            host: 'http://localhost:8080', // Local or cloud URL
            key: 'WEAVIATE_KEY' // optional for local deployment
        );
    }
}
```

### Elasticsearch

Elasticsearch's open source vector database offers an efficient way to create, store, and search vector embeddings. To use Elasticseach as a vector store in your agents implementation you have to import the official client:

```bash
composer require elasticsearch/elasticsearch
```

Here is how to create a RAG that uses Elasticsearch:

```php
namespace App\Neuron;

use Elastic\Elasticsearch\ClientBuilder;
use NeuronAI\RAG\RAG;
use NeuronAI\RAG\VectorStore\ElasticsearchVectorStore;
use NeuronAI\RAG\VectorStore\VectorStoreInterface;

class MyChatBot extends RAG
{
    ...

    protected function vectorStore(): VectorStoreInterface
    {
        $elasticsearch = ClientBuilder::create()
           ->setHosts(['<elasticsearch-endpoint>'])
           ->setApiKey('<api-key>')
           ->build();
       
        return new ElasticsearchVectorStore(
            client: $elasticsearch,
            index: 'neuron-ai'
        );
    }
}
```

### OpenSearch

Opensearch is the pure open source alternative to Elasticsearch. To use Opensearch in your agents you need to install its official client:

```bash
composer require opensearch-project/opensearch-php
```

Once you have the official client installed in your app you can return an instance of the `OpenSearchVectorStore` in your RAG agent:

```php
namespace App\Neuron;

use NeuronAI\RAG\RAG;
use NeuronAI\RAG\VectorStore\OpenSearchVectorStore;
use NeuronAI\RAG\VectorStore\VectorStoreInterface;
use OpenSearch\GuzzleClientFactory;

class MyChatBot extends RAG
{
    ...

    protected function vectorStore(): VectorStoreInterface
    {
        $opensearch = new GuzzleClientFactory()->create([
            'base_uri' => 'http://localhost:9200',
        ]);
        
        return new OpenSearchVectorStore(
            client: $opensearch,
            index: 'neuron-ai',
        );
    }
}
```

### Typesense

[Typesense](https://typesense.org/) is an open source alternative to the options above. To use Typesense in your agents you need to install its official client:

```bash
composer require typesense/typesense-php
```

Once you have the official client installed in your app you can return an instance of the TypesenseVectorStore in your RAG agent:

```php
namespace App\Neuron;

use NeuronAI\RAG\RAG;
use NeuronAI\RAG\VectorStore\TypesenseVectorStore;
use NeuronAI\RAG\VectorStore\VectorStoreInterface;

class MyChatBot extends RAG
{
    ...

    protected function vectorStore(): VectorStoreInterface
    {
        $typesense = new \Typesense\Client([
            'api_key' => 'TYPESENSE_API_KEY',
            'nodes' => [
                [
                    'host' => 'TYPESENSE_NODE_HOST',
                    'port' => 'TYPESENSE_NODE_PORT',
                    'protocol' => 'TYPESENSE_NODE_PROTOCOL'
                ],
            ]
        ]);
        
        return new TypesenseVectorStore(
            client: $typesense,
            collection: 'neuron-ai',
            vectorDimension: 1024
        );
    }
}
```

### Qdrant

[Qdrant](https://qdrant.tech/) is an open source vector database with strong similarity search capabilities. To use Qdrant in your agents you have to provide a `collectionUrl`. This means you will first need to create a collection on Qdrant with its attributes like: name, similarity search algorithm, vector dimension, etc.

Once you have the collection URL you can attach the `QdrantVectorStore` instance to your agent.

```php
namespace App\Neuron;

use NeuronAI\RAG\RAG;
use NeuronAI\RAG\VectorStore\QdrantVectorStore;
use NeuronAI\RAG\VectorStore\VectorStoreInterface;

class MyChatBot extends RAG
{
    ...

    protected function vectorStore(): VectorStoreInterface
    {
        return new QdrantVectorStore(
            collectionUrl: 'http://localhost:6333/collections/neuron-ai/',
            key: 'QDRANT_API_KEY'
        );
    }
}
```

### ChromaDB

[Chroma](https://trychroma.com/) is an open source database designed to be an AI application data source. To use ChromaDB in your agents you have to provide the name of an internal collection where you want to store the embeddings.

Once you have the collection created on your Chroma instance you can attach the `ChromaVectorStore` instance to the agent:

```php
namespace App\Neuron;

use NeuronAI\RAG\RAG;
use NeuronAI\RAG\VectorStore\ChromaVectorStore;
use NeuronAI\RAG\VectorStore\VectorStoreInterface;

class MyChatBot extends RAG
{
    ...

    protected function vectorStore(): VectorStoreInterface
    {
        return new ChromaVectorStore(
            collection: 'neuron-ai',
            //host: 'http://localhost:8000', <-- This is by default
            topK: 5
        );
    }
}
```

### Meilisearch

[Meilisearch](https://www.meilisearch.com/) is a hybrid search engine, but the Neuron implementation uses it exclusively as a vector store for embeddings and similarity search.

The `indexUid` parameter should be the identifier of a Meilisearch index that you have created and configured. Make sure this index defines a vector field whose dimension matches the embedding size produced by the embedder you are using. The `embedder` value (for example, `default`) must correspond to a named embedder configured in your Neuron setup so that the stored vectors and the index configuration stay aligned. Add the component to your RAG:

```php
namespace App\Neuron;

use NeuronAI\RAG\RAG;
use NeuronAI\RAG\VectorStore\MeilisearchVectorStore;
use NeuronAI\RAG\VectorStore\VectorStoreInterface;

class MyChatBot extends RAG
{
    ...

    protected function vectorStore(): VectorStoreInterface
    {
        return new MeilisearchVectorStore(
            indexUid: 'MEILISEARCH_INDEXUID',
            host: 'http://localhost:8000', // Or use the cloud URL
            key: 'MEILISEARCH_API_KEY',
            embedder: 'default',
            topK: 5
        );
    }
}
```

### MongoDB

To use this component you need to install the official mongoDB client:

```shellscript
composer require mongodb/mongodb
```

Use the component in you RAG or script:

```php
namespace App\Neuron;

use NeuronAI\RAG\RAG;
use NeuronAI\RAG\VectorStore\MongoDBVectorStore;
use NeuronAI\RAG\VectorStore\VectorStoreInterface;

class MyChatBot extends RAG
{
    ...

    protected function vectorStore(): VectorStoreInterface
    {
        $uri = 'mongodb://localhost:27017';
        $uriOptions = ['serverSelectionTimeoutMS' => 10000];
        $client = new MongoDB\Client($uri, $uriOptions);
        
        return new MongoDBVectorStore(
            client: $client,
            database: 'MONGODB_DATABASE',
            collectionName: 'MONGO_DB_COLLECTION',
            topK: 4
        );
    }
}
```

### Extend Vector Stores

If you want to support a new vector store you have to implement `VectorStoreInterface`:

```php
interface VectorStoreInterface
{
    public function addDocument(Document $document): VectorStoreInterface;

    /**
     * @param  Document[]  $documents
     */
    public function addDocuments(array $documents): VectorStoreInterface;

    /**
     * Delete every document matching the filters.
     */
    public function delete(FilterGroup $filters): VectorStoreInterface;

    /**
     * Return the documents most similar to the request's embedding.
     *
     * @return Document[]
     */
    public function search(SearchRequest $request): iterable;
}
```

There are two different methods for adding a single document or a collection of documents because many databases provide different APIs for these use cases. If the database you want to interact to doesn't handle these requests differently you can implement `addDocuments()` as a placeholder o `addDocument()`.

The `search()` method should return the list of documents with a similarity score not a similarity distance. If the underlying database returns a distance you can convert it to a score using the utility class `VectorSimilarity`:

```php
namespace App\Neuron\VectorStore;

use NeuronAI\RAG\Document;
use NeuronAI\RAG\VectorSimilarity;
use NeuronAI\RAG\VectorStore\VectorStoreInterface;

class MyVectorStore implements VectorStoreInterface
{
    ...


    /**
     * @param float[] $embeddings
     */
    public function search(SearchRequest $request): iterable
    {
        $documents = // get documents from the vector store

        return \array_map(function (Document $document) {
            return $document->setScore(
                VectorSimilarity::similarityFromDistance($similarity)
            );
        }, $documents);
    }
}
```

After creating your own implementation you can use it in the agent as any other store:

```php
namespace App\Neuron;

use App\Neuron\VectorStore\MyVectorStore;
use NeuronAI\Agent;
use NeuronAI\RAG\VectorStore\VectorStoreInterface;

class MyAgent extends Agent
{
    protected function vectorStore(): VectorStoreInterface
    {
        return new MyVectorStore(
            key: 'VECTORSTORE_API_KEY',
            index: 'neuron-ai',
        );
    }
}
```

## Filters

### DocumentSchema

`Document` is the unified processing object across loading, splitting, embedding, storage, retrieval, middleware, and reranking. Its fields are accessed through methods. Embedding and score are nullable runtime values; strict `null` checks express whether a stage produced them (`0.0` remains a valid score).

Custom metadata stays schema-less for storage and round-tripping. Portable filtering requires a collection-level schema passed to the vector store:

```php
$schema = DocumentSchema::of(
    DocumentField::string('tenant')->required()->filterable(),
    DocumentField::integer('year')->filterable(),
    DocumentField::strings('tags')->filterable(),
);

$store = new FileVectorStore(schema: $schema);
```

Stores validate declared values and filters locally. Only `sourceType`, `sourceName`, and declared filterable metadata fields are portable filter targets. Array fields are supported for validation/storage but need raw backend filters except for portable filterable string arrays, which support `containsAny` and `containsAll`. Declared arrays must be non-empty homogeneous lists. A `DocumentField` can be passed directly to filter factories for schema-aware construction. `neq` requires a required field so missing-field behavior cannot diverge between databases. RAG validates documents before embedding.

### Filter Expression

Once you have deined ilterable fields with `DocumentSchema`, you can pass a filtering expression to the retriaval strategy at runtime.

Filter are a portable, backend-neutral expression tree compiled to each store's native syntax. This is **filtered similarity search**: metadata filters constrain vector similarity results. Reserve **hybrid search** for strategies that combine vector and lexical ranking.

```php
class MyRAG extends RAG
{
    ...,
    
    protected function retrieval(): RetrievalInterface
    {
        return new SimilarityRetrieval(
            vectorStore: $this->resolveVectorStore(),
            embeddingProvider: $this->resolveEmbeddingsProvider(),
            filters: Filter::gt('year', 2020)
                ->lt('year', date('Y'))
                ->eq('tenant', Tenant::id())
        );
    }
}
```

### Available Filters

**`Filter::eq/neq/in/gt/gte/lt/lte(field, value)`** — low-level comparison factories.

**`Filter::where(field, value)`** starts an immutable fluent `Criteria` with `where*` methods for the common path. Values are scalars only (`null` throws: no portable missing-vs-null semantics); range values normalize to `int|float` (string ranges are not portable). Backed enums normalize to their value and `DateTimeInterface` values normalize to epoch timestamps.

**`FilterGroup::allOf(...)` / `anyOf(...)`** — nested boolean expressions; `and(...)` / `or(...)` remain short aliases. Same-operator groups flatten, while mixed operators preserve their boundaries.

**`Filter::containsAny/containsAll(field, values)`** — portable filtering for filterable `string[]` fields. Other array types remain backend-native.

**`Filter::raw(StoreClass::class, $fragment)`** — backend-native escape hatch, tagged with its target store. The tagged store passes the fragment through verbatim; every other store's compiler throws (fail-loud on store swap, never silent misfiltering). Raw fragments must be trusted, developer-authored syntax; never interpolate request values into them.

**`FilterScope::merge(...)`** — combines independently supplied mandatory scopes with a root AND. Query expressiveness and scope safety are separate.

**`FilterGroup::allOf(...)` / `anyOf(...)`** — nested boolean expressions; `and(...)` / `or(...)` remain short aliases. Same-operator groups flatten, while mixed operators preserve their boundaries.
