---
description: Evaluating the output of your agentic system
---

# Evaluation

This guide covers approaches to evaluating agents. Effective evaluation is essential for measuring agent performance, tracking improvements, and ensuring your agents meet quality standards.

When building AI agents, evaluating the consistency of their output is crucial during this process. It's important to consider various qualitative and quantitative factors, including response syntax, task completion, success, and inaccuracies or hallucinations. In evaluations, it's also important to consider comparing different agent configurations to optimize for specific desired outcomes. Given the dynamic and non-deterministic nature of LLMs, it's also important to have rigorous and frequent evaluations to ensure a consistent baseline for tracking improvements or regressions.

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

Next create the `evaluators` directory in your project root folder. Keeping test code separate from production code creates a clear boundary between what gets deployed to production and what exists purely for development and quality assurance.

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

### Defining The Dataset Loader

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

### Output Interfaces

The evaluation module uses a PHP configuration file to control how evaluation results are output. The config system supports multiple output drivers, enabling results to be sent to console, files, databases, or external APIs simultaneously.

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
