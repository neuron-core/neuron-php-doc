---
description: Evaluating the output of your agentic system
---

# Evals

This guide covers approaches to evaluating agents. Effective evaluation is essential for measuring agent performance, tracking improvements, and ensuring your agents meet quality standards.

When building AI agents, evaluating the consistency of their output is crucial, not only for the maintenance of the agent, but also to evaluate different architectures or prompting approach on the initial design phase.&#x20;

It's important to consider various qualitative and quantitative factors, including response syntax, task completion, success, and inaccuracies or hallucinations. In evaluations, it's also important to consider comparing different agent configurations to optimize for specific desired outcomes. Given the dynamic and non-deterministic nature of LLMs, it's also important to have rigorous and frequent evaluations to ensure a consistent baseline for tracking improvements or regressions.

### Configuring your application

Like unit tests, it could be better to collect evaluators for your AI system into a dedicated directory. So, you can add the configuration below to your application `composer.json` file in order to tell composer how to include your evaluators in the application namespaces:

```json
"autoload-dev": {
    "psr-4": {
        ...,
        "App\\Evaluators\\": "evaluators/"
    }
},
```

Next create the `evaluators` directory in your project root folder. Keeping evaluation code separate from production code creates a clear boundary between what gets deployed to production and what exists purely for development and quality assurance.

### Creating Evaluators

Use the command below to create the `AgentEvaluator` class into the evaluators folder:

{% tabs %}
{% tab title="Unix" %}
```bash
vendor/bin/neuron make:evaluator App\\Neuron\\Evaluators\\AgentEvaluator
```
{% endtab %}

{% tab title="Windows" %}
```powershell
.\vendor\bin\neuron make:evaluators App\Neuron\Evaluators\AgentEvaluator
```
{% endtab %}
{% endtabs %}

The class being created will have the following structure:

```php
namespace App\Neuron\Evaluators;

use NeuronAI\Evaluation\Assertions\StringContains;
use NeuronAI\Evaluation\BaseEvaluator;
use NeuronAI\Evaluation\Contracts\DatasetInterface;
use NeuronAI\Evaluation\Dataset\JsonDataset;

class AgentEvaluator extends BaseEvaluator
{
    /**
     * 1. Get the dataset to evaluate against
     */
    public function getDataset(): DatasetInterface
    {
        return new JsonDataset(__DIR__ . '/datasets/dataset.json');
    }

    /**
     * 2. Run the agent logic being tested
     */
    public function run(array $datasetItem): mixed
    {
        $response = MyAgent::make()->chat(
            new UserMessage($datasetItem['input'])
        )->getMessage();
        
        return $response->getContent();
    }

    /**
     * 3. Evaluate the output against expected results, with assertions
     */
    public function evaluate(mixed $output, array $datasetItem): void
    {
        $this->assert(
            new StringContains($datasetItem['reference']),
            $output,
        );
    }
} 
```

The logic is quite straightforward. The evaluator first load the dataset, and then run the evaluation for each item of the dataset.

In the `run` method you can execute your agentic entities with the example input and return the output. The output is then passed to the `evaluate` method where you can performs assetions comparing the output with a reference value or any other logic you want.

### Dataset Loader

You can use anything you want as dataset. There are no predefined format. The evaluator class simply allows you to load a list of test cases and run the evaluators against them. You have two dataset loaders.

#### ArrayDataset

```php
class AgentEvaluator extends BaseEvaluator
{
    public function getDataset(): DatasetInterface
    {
        return new ArrayDataset([
            [
                'input' => 'Hi',
                'reference' => 'help'
            ]
        ]);
    }
    
    ...
}
```

#### JsonDataset

```php
class AgentEvaluator extends BaseEvaluator
{
    public function getDataset(): DatasetInterface
    {
        return new JsonDataset(__DIR__ . '/datasets/dataset.json');
    }
    
    ...
}
```

You can eventually create a custom dataset loader implementing `NeuronAI\Evaluation\Contracts\DatasetInterface`.

### Running Evaluations

If you have properly configured your composer file you can use the Neuron CLI to launch the evaluators:

{% tabs %}
{% tab title="Unix" %}
```bash
vendor/bin/neuron evaluations --path=evaluators
```
{% endtab %}

{% tab title="Windows" %}
```powershell
.\vendor\bin\neuron evaluations --path=evaluators
```
{% endtab %}
{% endtabs %}

### Assertions

We provide a set of built-in assertion for the most common use case. You can also implement your own assertion to design custom scoring systems. Check the next section.

**StringContains**

```php
$this->assert(new StringContains('positive'), $output);
```

**StringContainsAll**

Check if the output contains all keywords:

```php
$this->assert(new StringContainsAll(['hello', 'world']), $output);
```

**StringContainsAny**

Check if the output contains any of the keywords:

```php
$this->assert(new StringContainsAny(['success', 'completed']), $output);
```

**StringStartsWith**

Check if the output starts with a prefix:

```php
$this->assert(new StringStartsWith('Hello'), $output);
```

**StringEndsWith**

Check if the output ends with a suffix:

```php
$this->assert(new StringEndsWith('!'), $output);
```

**StringLengthBetween**

Check if the string length is within range:

```php
$this->assert(new StringLengthBetween(10, 100), $output);
```

**StringDistance**

Check string similarity using Levenshtein distance:

```php
$this->assert(new StringDistance(
    reference: 'expected text',
    threshold: 0.5, // Minimum similarity score
    maxDistance: 50 // Maximum allowed edits
), $output);
```

**StringSimilarity**

Check string similarity using embeddings:

```php
use NeuronAI\Evaluation\Assertions\StringSimilarity;
use NeuronAI\RAG\Embeddings\OpenAI\OpenAIEmbeddings;

$this->assert(new StringSimilarity(
    reference: 'The quick brown fox',
    embeddingsProvider: new OpenAIEmbeddings(key: 'YOUR_KEY'),
    threshold: 0.6
), $output);
```

**MatchesRegex**

Match against regular expression:

```php
$this->assert(new MatchesRegex('/^\d{3}-\d{2}-\d{4}$/'), $output);
```

**IsValidJson**

Check if the output is valid JSON:

```php
$this->assert(new IsValidJson(), $output);
```

### AI as a Judge

Use an AI agent to evaluate outputs with custom criteria. Neuron provdes you with the primitive class AgentJudge to define your custom creteria, otherwise you can use one of the built-in judge assertions.

```php
use NeuronAI\Evaluation\Assertions\AgentJudge;

class AgentJudgeEvaluator extends BaseEvaluator
{
    protected AgentInterface $judge;

    public function setUp(): void
    {
        $this->judge = Agent::make()
            ->setAiProvider(
                new Antrhopic(...)
            )
            ->setInstructions('You are an expert evaluator for customer support responses.');
    }

    public function getDataset(): DatasetInterface
    {
        return new JsonDataset(...);
    }

    public function run(array $datasetItem): mixed
    {
        $response = MyAgent::make()->chat(
            new UserMessage($datasetItem['input'])
        )->getMessage();
        
        return $response->getContent();
    }
    
    public function evaluate(mixed $output, array $datasetItem): void
    {
        $this->assert(new AgentJudge(
            judge: $this->judge,
            criteria: 'Response should be helpful, polite, and address the customer\'s question directly',
            threshold: $datasetItem['threshold']
        ), $output);
    }
}
```

#### Faithfulness Judge

Check if output is grounded in context (no hallucinations):

```php
$this->assert(new FaithfulnessJudge(
    judge: $this->judge,
    context: $retrievedDocuments,
    threshold: 0.7
), $output);
```

#### Correctness Judge

Compare to expected answer:

```php
$this->assert(new CorrectnessJudge(
    judge: $judge,
    expected: $datasetItem['expected_answer'],
    threshold: 0.7
), $output);
```

#### Relevance Judge

Check if output addresses the question:

```php
$this->assert(new RelevanceJudge(
    judge: $judge,
    question: $datasetItem['question'],
    threshold: 0.7
), $output);
```

#### Helpfulness Judge

Evaluate utility and actionability:

```php
$this->assert(new HelpfulnessJudge(
    judge: $judge,
    threshold: 0.7
), $output);
```

### Creating Custom Assertions

```php
use NeuronAI\Evaluation\Assertions\AbstractAssertion;
use NeuronAI\Evaluation\AssertionResult;

class GreaterThanAssertion extends AbstractAssertion
{
    public function __construct(
        private readonly float $threshold
    ) {}

    public function evaluate(mixed $actual): AssertionResult
    {
        if (!is_numeric($actual)) {
            return AssertionResult::fail(
                0.0,
                'Expected numeric value, got ' . gettype($actual),
            );
        }

        if ($actual > $this->threshold) {
            return AssertionResult::pass(1.0);
        }

        return AssertionResult::fail(
            0.0,
            "Expected {$actual} to be greater than {$this->threshold}",
        );
    }
}
```

### Output

The evaluation module uses a PHP configuration file to control how evaluation results are displayed. The config system supports multiple output drivers, enabling results to be sent to console, files, databases, or external APIs simultaneously.

#### **Config File**

Create the `evaluation.php` file in your project root:

```php
<?php

use NeuronAI\Evaluation\OutputDrivers\ConsoleDriver;
use NeuronAI\Evaluation\OutputDrivers\JsonDriver;

return [
    'output' => [
        // Output results in the console
        ConsoleDriver::class => ['verbose' => true],

        // Save results in a json file
        JsonDriver::class => ['path' => 'evaluation-results.json'],
    ],
];
```

You can declare an array of options for each output class. This configurations will be passed as arguments to the constructor of the output class implementation.

**If no config file exists**, the system defaults to `ConsoleOutputDriver` with standard output.

#### Creating Custom Output

Implement `EvaluationOutputInterface` to create custom output drivers:

```php
namespace App\Neuron\Evaluations;

use NeuronAI\Evaluation\Contracts\EvaluationOutputInterface;
use NeuronAI\Evaluation\Runner\EvaluatorSummary;

class DatabaseOutput implements EvaluationOutputInterface
{
    public function __construct(
        private readonly \PDO $pdo,
        private readonly string $table = 'evaluations'
    ) {}

    public function output(EvaluatorSummary $summary): void
    {
        $stmt = $this->pdo->prepare(
            "INSERT INTO {$this->table} (passed, failed, success_rate, total_time, created_at, updated_at) VALUES (?, ?, ?, ?, NOW(), NOW())"
        );
        $stmt->execute([
            $summary->getPassedCount(),
            $summary->getFailedCount(),
            $summary->getSuccessRate(),
            $summary->getTotalExecutionTime(),
        ]);
    }
}
```

Once you have created your output class you can register it in the configuration file, to be used the next time you run the evaluations.

```php
<?php

use NeuronAI\Evaluation\OutputDrivers\ConsoleDriver;
use NeuronAI\Evaluation\OutputDrivers\JsonDriver;

return [
    'output' => [
        // Output results in the console
        ConsoleDriver::class => ['verbose' => true],

        // Save results in a json file
        //JsonDriver::class => ['path' => 'evaluation-results.json'],
        
        // Save results in the database
        DatabaseOutput::class => [
            'pdo' => new \PDO(...),
            'table' => 'evaluations',
        ]
    ],
];
```
