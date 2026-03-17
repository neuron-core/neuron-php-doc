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

### Creating Evaluator

Use the command below to create the `AgentEvaluator` class into the evaluators folder:

{% tabs %}
{% tab title="Unix" %}
```bash
vendor/bin/neuron make:evaluator App\\Evaluators\\AgentEvaluator
```
{% endtab %}

{% tab title="Windows" %}
```powershell
.\vendor\bin\neuron make:evaluators App\Evaluators\AgentEvaluator
```
{% endtab %}
{% endtabs %}

The class being created will have the following structure:

```php
namespace App\Evaluators;

use NeuronAI\Evaluation\Assertions\StringContains;
use NeuronAI\Evaluation\BaseEvaluator;
use NeuronAI\Evaluation\Contracts\DatasetInterface;
use NeuronAI\Evaluation\Dataset\JsonDataset;

class AgentEvaluator extends BaseEvaluator
{
    public function getDataset(): DatasetInterface
    {
        return new JsonDataset(__DIR__ . '/datasets/dataset.json');
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
        $this->assert(
            new StringContains($datasetItem['reference']),
            $output,
        );
    }
} 
```

The logic is quite straightforward. The evaluator first load the dataset, and then run the evaluation for each item of the dataset.

In the `run` method you can execute your agentic entities with the example input and return the output. The output is then passed to the `evaluate` method where you can performs assetions comparing the output with a reference value or any other logic you want.

### Defining The Dataset

You can use anything you want as dataset. There are no predefined format. The evaluator class simply allows you to load a list of test cases to run the evaluators against them. You have two dataset reader.

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

### Running Evaluations

If you have properly configured your composer file you can use the Neuron CLI to launch the evaluators:

```bash
./vendor/bin/neuron evaluations --path=evaluators
```
