# Upgrade Guide

### Updating Dependencies

You should update the following dependencies in your application's `composer.json` file:

* **inspector-apm/neuron-ai** to **^2.0**

### High Impact Changes

#### Workflow

The Workflow component was completely re-architected. The Edge class was completely removed and now the orchestration system is event-driven relying only on node definition.

It also supports real-time streaming.

We strongly recommend to use the V2 documentation to understand the new system and move your previously created Workflow to the new architecture.

<a href="broken-reference" class="button secondary" data-icon="arrow-right-long">Workflow Documentation</a>

### Low Impact

#### RAG Retrieval component

In the previous version the RAG component could interact with a vector store and an embeddings provider, but there was no way to customize this behavior. Recently many different retrieval techniques emerged trying to increase the accuracy of a RAG system.

Neuron RAG now has a separate retrieval component that allwos you to implement different strategies to accomplish context retrieval from an external data source. By default RAG uses `SimilarityRetrieval` that simply replicate the previous behaviour maintaining backward compatibility. But it depends now by its own interface so you can create custom implementation and inject it into the RAG.

<a href="upgrade-guide.md#rag-retrieval-component" class="button secondary" data-icon="arrow-right-long">Retrieval Documentation</a>

### New Features

#### Neuron CLI

V2 ships with practical developer experience improvements that address common friction points. The CLI tool brings the new “make” command that helps you scaffold common classes reducing boilerplate fatigue:

```bash
# Unix
php vendor/bin/neuron make:agent App\\Neuron\\MyAgent

# Windows
php .\vendor\bin\neuron make:agent App\\Neuron\\MyAgent
```

#### Evaluators

When building AI agents, evaluating their performance is crucial during this process. It's important to consider various qualitative and quantitative factors, including response quality, task completion, success, and inaccuracies or hallucinations.

Neuron introduces a system to create evaluators against test cases, so you can continues verify the output of your agentic entities overtime.

<a href="../../components/evaluation.md" class="button secondary" data-icon="arrow-right-long">Evaluation Documentation</a>

#### Structured Output validation rules

We introduced two new validation rules for structured output: [WordsCount](upgrade-guide.md#structured-output-validation-rules) (works on string), and [InRange](upgrade-guide.md#structured-output-validation-rules) (works on numeric).

#### Tool Max Tries

Agents now have a safety mechanism that tracks the number of times a tool is invoked during an execution session. If the agent exceeds this limit, execution is interrupted and an exception is thrown. By default the limit is 5 calls, and it count for each tool individually.&#x20;

You can customize this value with the `toolMaxTries()` method at agent level, or use `setMaxTries()` on the tool level. Setting Max tries on single tool takes precedence over the global setting.

```php
try {

    $result = YouTubeAgent::make()
        ->toolMaxTries(5) // Max number of calls for each tool
        ->addTool(
            // It takes precedence over the global setting
            CustomTool::make()->setMaxTries(2)
        )
        ->chat(...);
        
} catch (ToolMaxTriesException $exception) {
    // do something
}
```

