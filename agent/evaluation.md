---
description: Evaluating output and behaviour of your agentic system
metaLinks:
  alternates:
    - https://app.gitbook.com/s/GHx4l2LknIex7vFIUg1R/agent/evaluation
---

# Evaluations

{% hint style="warning" %}
#### Coding Agent Skill

Use **`/neuron-evaluation`** to teach your coding agent how to implement a complete suite of evaluations for your agentic entities.

[AI-Assisted Development](../overview/agentic-development.md)
{% endhint %}

This guide covers approaches to evaluating agents. Effective evaluation is essential for measuring agent performance, tracking improvements, and ensuring your agentic system meet quality standards.

It's important to consider various qualitative and quantitative factors, including response syntax, task completion, success, and inaccuracies or hallucinations. In evaluations, it's also important to consider comparing different configurations to optimize for specific desired outcomes. Given the dynamic and non-deterministic nature of LLMs, it's also important to have rigorous and frequent evaluations to ensure a consistent baseline for tracking improvements or regressions.

### Why You Need Evaluations

Evals play a role that goes beyond the technical practice, and that has no equivalent in traditional software development. An agent's quality is probabilistic: there is no binary "it works" to point at, so the question "is it ready?" has no obvious answer. A strong evaluation suite becomes that answer, a shared, objective definition of "good enough" that you build together with your customer or stakeholders. When they ask how the project is going, you don't reply with anecdotes and demo impressions: you show the success rate on a dataset everyone agreed represents the real use cases, and how it moved since the last release. When a customer reports a bad interaction, that interaction becomes a new dataset item — a complaint turned into a measurable test case that can never silently regress again.

This is something profoundly different from unit tests. Unit tests never had this role: they are an internal engineering tool, invisible outside the team, no one ever discussed a project's status with a customer by looking at a test suite. Evals face outward. They are the instrument you use to negotiate expectations while iterating, to manage the relationship with the people relying on the system, and to prove, release after release, that the agent is getting better at the things that matter to them.

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

### Custom bootstrap file

By default, the `neuron` CLI only loads your project's Composer autoloader (`vendor/autoload.php`). That's enough when your classes are plain PHP resolvable by Composer, but often they aren't: an evaluator might read config through your framework's helpers, need environment variables loaded from `.env`, rely on constants, or require a service container to be initialized. In those cases the command would fail with "class not found" or missing-configuration errors, because the code that normally prepares that environment — your framework's bootstrap — never runs.

The `--autoload-file` option solves this by letting you point the CLI to a PHP file to execute _before_ the command starts, in addition to the default Composer autoloader:

```bash
vendor/bin/neuron evaluation /path/to/evaluators --autoload-file=bootstrap.php
```

The file can do anything a normal bootstrap does — register additional autoloaders, load environment variables, define constants, or boot your framework. For example, to run evaluators that depend on a Laravel application:

```php
<?php
// bootstrap.php

require __DIR__.'/vendor/autoload.php';

$app = require_once __DIR__.'/bootstrap/app.php';
$app->make(Illuminate\Contracts\Console\Kernel::class)->bootstrap();
```

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
        ConsoleDriver::class,
        // new ConsoleDriver(verbose: true),

        // Save results in a json file
        new JsonDriver(path: 'evaluation-results.json'),
    ],
];
```

You can provide a class-string for classes with zero-args constructor or directly a concrete instance.

**If no config file exists**, the system defaults to `ConsoleDriver` with standard output.

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
        new ConsoleDriver(verbose: true),

        // Save results in a json file
        //new JsonDriver(path: 'evaluation-results.json'),
        
        // Save results in the database
        new DatabaseOutput(
            pdo: new \PDO(...),
            table: 'evaluations',
        )
    ],
];
```

### Parallel Evaluations

By default the evaluation command processes dataset items one at a time. Since most evaluators spend their time waiting on AI provider responses, you can drastically reduce the total run time by processing multiple dataset items in parallel with the `--concurrency` option:

```bash
vendor/bin/neuron evaluation path/to/evaluators --concurrency=3
```

With `--concurrency=3`, up to 3 dataset items are evaluated at the same time, each in its own PHP child process. An evaluation that makes one 2-second LLM call per item over a 100-item dataset drops from \~200 seconds to \~66 seconds.

#### Requirements

Parallel execution relies on process forking, which requires:

* The [pcntl](https://www.php.net/manual/en/book.pcntl.php) PHP extension (available on Linux and macOS — not on Windows)
* The [spatie/fork](https://github.com/spatie/fork) package:

```bash
composer require --dev spatie/fork
```

If either is missing, the command prints a notice and automatically falls back to sequential execution, so the same command works in every environment.

#### Choosing a concurrency level

Every item in flight is an active request against your AI provider. Start with a moderate value (3–5) and increase it as long as you don't hit provider rate limits. If you see rate limit errors appearing as test failures, lower the value.

#### How it works, and what to watch out for

Each dataset item runs in a forked copy of your evaluator, and its result is sent back to the parent process. This has a few practical implications:

* **Results are unaffected.** Items are evaluated independently, results keep their dataset order, and the final report is identical to a sequential run.
* **State is not shared between items.** Each item sees the evaluator state as it was after `setUp()`. Side effects performed while handling one item (incrementing a property, appending to a file) are not visible to other items. If your evaluator relies on accumulating state across items, keep running it sequentially.
* **Outputs must be serializable.** The value returned by `run()` crosses a process boundary via `serialize()`. If it can't be serialized (e.g. it contains a closure or an open connection), the assertion results are preserved but the output shown in reports is replaced with a placeholder string.

#### Execution time reporting

The reported total time is the real wall-clock duration of the run, while the average time per test reflects the actual duration of each individual item — so under parallel execution the average per test can be larger than the total divided by the number of tests.

## Evaluating Multi-Turn Conversations

The evaluators you have seen so far follow a simple pattern: run your agent once, get an output, assert against it. This works well for single-shot tasks, but it doesn't reflect how agentic applications actually work. An AI Agent produces its value across a whole conversation: it gathers information over multiple turns, decides which tools to call and with which arguments, asks a human for approval before dangerous actions, and recovers when that approval is denied.

Evaluating only the final response misses most of this behavior. The agent may have given a perfectly polite answer while calling the wrong tool, skipping a required lookup, or executing an action it should have submitted for approval first.

Neuron provides two components to close this gap, and one sentence captures how they relate: **you run a Conversation; you evaluate its Trajectory.**

* `Conversation` is the execution helper. It drives your agent through a multi-turn exchange inside the evaluator's `run()` method — delivering user turns, answering approval requests, or even letting another AI play the user.
* `Trajectory` is the recorded subject. It wraps the conversation's messages and answers evaluation questions about what actually happened: which tools were called, with which arguments, what was approved or rejected, and what the agent finally said.

### Running a Conversation

Use `Conversation` inside `run()` to script a multi-turn exchange. Each turn is delivered only after the previous one fully completed:

```php
use NeuronAI\Evaluation\BaseEvaluator;
use NeuronAI\Evaluation\Contracts\DatasetInterface;
use NeuronAI\Evaluation\Conversation\Conversation;
use NeuronAI\Evaluation\Dataset\JsonDataset;

class RefundConversationEvaluator extends BaseEvaluator
{
    public function getDataset(): DatasetInterface
    {
        return new JsonDataset(__DIR__ . '/datasets/refunds.json');
    }

    public function run(array $datasetItem): mixed
    {
        return Conversation::make(MyAgent::make())
            ->withTurns($datasetItem['turns'])
            ->run();
    }

    public function evaluate(mixed $trajectory, array $datasetItem): void
    {
        // Assert against the returned Trajectory (next sections)
    }
}
```

The turns are a simple list, so they can live directly in your JSON dataset:

```json
[
    {
        "turns": [
            "Hi, I want a refund for order #123",
            "Yes, please proceed."
        ]
    }
]
```

Entries can be plain strings or `UserMessage` instances, so you can also script turns carrying images or documents.

### The Trajectory

A chat history is a raw list of messages, and asserting against it directly means writing the same boilerplate in every evaluator: matching tool calls to their results, reading approval states, extracting the final answer. The `Trajectory` returned by `Conversation::run()` does this work once and exposes what you actually want to ask:

```php
$trajectory->toolCalls();                // every tool call, in execution order
$trajectory->toolCalls('refund_order'); // filtered by tool name
$trajectory->lastToolCall();            // the most recent call (or null)
$trajectory->finalAnswer();             // the agent's last reply ('' if none)
$trajectory->userMessages();            // the user side of the conversation
$trajectory->usage();                   // aggregate token usage of the whole conversation
$trajectory->toTranscript();            // human-readable transcript (what AI judges read)
$trajectory->messages();                // the underlying messages, full fidelity
```

`toolCalls()` returns the framework's own `ToolInterface` objects, so you inspect them with the API you already know — `getInputs()`, `getResult()`, `getApprovalState()`. When a call appears twice in the history (the pending snapshot when the agent proposed it, and the final outcome after it ran), the Trajectory merges them into one entry, with the final outcome winning.

You don't need the `Conversation` helper to get a Trajectory. If you drive the agent yourself — a custom loop, streaming, a bespoke workflow — you can project any chat history and use the same assertions:

```php
use NeuronAI\Evaluation\Trajectory\Trajectory;

$trajectory = Trajectory::fromChatHistory($agent->getChatHistory());
```

Since `usage()` aggregates the provider-reported token counts across every turn, you can also keep an eye on cost regressions as part of your evaluation suite:

```php
$this->assert(
    new GreaterThanAssertion($datasetItem['token_budget']),
    $trajectory->usage()->getTotal()
);
```

### Trajectory Assertions

These assertions answer the questions that final-output checks can't: did the agent take the right path to the answer?

#### **ToolWasCalled**

Verify the agent invoked a tool, optionally constraining its arguments. The array form is a subset match — every listed key must be present and equal, extra arguments are allowed:

```php
use NeuronAI\Evaluation\Assertions\Trajectory\ToolWasCalled;

// The tool was called, with any arguments
$this->assert(new ToolWasCalled('search_orders'), $trajectory);

// The tool was called with these arguments
$this->assert(
    new ToolWasCalled('refund_order', ['order_id' => '123']),
    $trajectory
);

// Full control with a callable
$this->assert(
    new ToolWasCalled('refund_order', fn (array $inputs): bool => $inputs['amount'] <= 100),
    $trajectory
);
```

#### **ToolWasNotCalled**

The guardrail assertion — verify the agent did _not_ take a forbidden path:

```php
use NeuronAI\Evaluation\Assertions\Trajectory\ToolWasNotCalled;

$this->assert(new ToolWasNotCalled('delete_account'), $trajectory);
```

#### **TrajectoryMatches**

Verify the sequence of tool calls against an expected list of tool names. The `Mode` enum decides how strict the comparison is:

```php
use NeuronAI\Evaluation\Assertions\Trajectory\Mode;
use NeuronAI\Evaluation\Assertions\Trajectory\TrajectoryMatches;

$this->assert(
    new TrajectoryMatches(['search_orders', 'refund_order'], Mode::Subset),
    $trajectory
);
```

* `Mode::Strict` — exactly these calls, in this order, nothing else. Best for pinning down a regression baseline.
* `Mode::Unordered` — the same calls, in any order.
* `Mode::Subset` — the expected calls appear in this order, extra calls are allowed in between. Best when you care about the essential path but tolerate incidental calls.
* `Mode::Superset` — no call outside the expected set. The expected list acts as an allow-list.

The expected list is plain strings, so it can live in your dataset next to the turns.

#### **ToolWasApproved / ToolWasRejected**

Verify the human decision recorded on a tool call (see the next section for how those decisions are produced during an evaluation):

```php
use NeuronAI\Evaluation\Assertions\Trajectory\ToolWasApproved;
use NeuronAI\Evaluation\Assertions\Trajectory\ToolWasRejected;

$this->assert(new ToolWasApproved('send_email'), $trajectory);
$this->assert(new ToolWasRejected('refund_order'), $trajectory);
```

Notice there are no dedicated assertions for the final answer: everything you learned in the previous sections still applies. Pass `$trajectory->finalAnswer()` to the string assertions, or to an AI judge:

```php
$this->assert(new StringContains('cannot process'), $trajectory->finalAnswer());
```

### Evaluating Human-In-The-Loop

If your agent uses the `ToolApproval` middleware, `chat()` doesn't return an answer when a gated tool is requested — it suspends, waiting for a human decision. In production a person sees the pending action and approves or rejects it. In an evaluation there is no person, so without help the conversation would simply stall.

At the same time, these are exactly the flows you most want to test: does the agent request approval before dangerous actions? Does it recover gracefully when the human says no?

The `withApprovals()` method lets you script the approver. The callable you pass is invoked whenever the agent suspends — at any point in the conversation — and returns the decisions:

```php
use NeuronAI\Evaluation\Trajectory\Trajectory;
use NeuronAI\Workflow\Interrupt\ApprovalRequest;

Conversation::make($agent)
    ->withTurns($datasetItem['turns'])
    ->withApprovals(function (ApprovalRequest $request, Trajectory $soFar): array {
        $payload = [];

        foreach ($request->getActions() as $action) {
            if ($action->isPending()) {
                $payload[$action->id] = 'approve';   // or ['reject', 'the reason']
            }
        }

        return $payload;
    })
    ->run();
```

It is a separate method — not an entry in the turns script — because you can't know in advance at _which_ turn the model will decide to call the gated tool. The policy is positionless: it answers whenever the suspension happens.

The second argument gives you the conversation so far, which is how you write decisions that depend on the tool's arguments:

```php
->withApprovals(function (ApprovalRequest $request, Trajectory $soFar): array {
    $payload = [];

    foreach ($request->getActions() as $action) {
        $args = $soFar->lastToolCall($action->name)?->getInputs();

        $payload[$action->id] = ($args['amount'] ?? 0) > 100
            ? ['reject', 'Above the auto-approve threshold']
            : 'approve';
    }

    return $payload;
})
```

Two rules keep evaluations honest, both enforced with an `EvaluationException` that marks the dataset item as an error:

* If the agent suspends and no policy is configured, the evaluation fails loudly. There is deliberately no "approve everything" default — that would silently fake the exact human behavior you are measuring.
* The returned payload must contain a decision for every pending action. Neuron's approval system never treats silence as consent, and an incomplete decision set would leave the workflow suspended forever.

The rejection, the reason delivered to the model, and the agent's recovery all end up in the Trajectory — where `ToolWasRejected` and the string assertions on `finalAnswer()` can verify them.

### Simulated Users

A scripted list of turns has a limit: it can't react to what the agent actually says. If the agent asks a clarifying question your script didn't anticipate, the next scripted turn will read as a non-sequitur. Real users adapt — and testing how your agent handles an adaptive counterpart requires one.

`UserSimulator` is an AI agent that plays the user. You give it a persona and a goal, and at every step it reads the conversation so far and decides: continue with a new message, or stop because the goal is satisfied (or because it's giving up):

```php
use NeuronAI\Evaluation\Conversation\UserSimulator;

$simulator = UserSimulator::make()
    ->withPersona('An impatient customer who gives short answers')
    ->withGoal('Get a refund for order #123');

$simulator->setAiProvider(new Anthropic(key: '...', model: '...'));

$trajectory = Conversation::make($agent)
    ->withUser($simulator, maxTurns: 10)
    ->run();
```

`withUser()` replaces `withTurns()` — a conversation is either scripted or simulated, not both. A few things to know:

* `maxTurns` is required. It is a hard cap on the number of user turns, and there is no infinite default. Hitting the cap ends the conversation _normally_: whether an unfinished conversation counts as a failure is a judgment for your assertions (the `TaskCompletionJudge` below is the natural fit), not for the runner.
* The simulator decides on its own when the goal is reached — no separate referee model is involved.
* The simulator never answers approval requests. In production the user chatting with your agent and the human approving its actions are different people, and the evaluation preserves that separation: approvals stay with `withApprovals()`, in simulated conversations too.

Since `UserSimulator` is a regular Neuron Agent, you can back it with any provider — including a cheaper model than the one under test.

### Judging the Whole Conversation

The AI judges you saw earlier evaluate a single piece of text. But some questions can only be answered by looking at the entire exchange: did the agent actually accomplish what the user came for, considering the tools it ran, the approvals it was denied, and everything it said along the way?

All judges now accept a `Trajectory` directly. When you pass one, the judge receives the full conversation transcript — user and assistant turns, tool calls with their arguments and results, and approval decisions with their reasons:

```php
use NeuronAI\Evaluation\Assertions\Judges\TaskCompletionJudge;

$this->assert(
    new TaskCompletionJudge(
        judge: $this->judge,
        goal: $datasetItem['goal'],
        threshold: 0.7
    ),
    $trajectory
);
```

`TaskCompletionJudge` is calibrated for exactly this question. It scores goal completion over the whole trajectory, and it knows how to weigh human-in-the-loop outcomes: an agent whose action was rejected by the approver is judged on how it handled the rejection, not punished for the denial itself.

The generic `AgentJudge` and the other pre-configured judges accept a Trajectory the same way, so you can evaluate tone, helpfulness, or custom criteria against the full conversation instead of a single message.

### Putting It All Together

A complete evaluator for a refund scenario with human approval:

```php
use NeuronAI\Evaluation\Assertions\Judges\TaskCompletionJudge;
use NeuronAI\Evaluation\Assertions\StringContains;
use NeuronAI\Evaluation\Assertions\Trajectory\Mode;
use NeuronAI\Evaluation\Assertions\Trajectory\ToolWasRejected;
use NeuronAI\Evaluation\Assertions\Trajectory\TrajectoryMatches;
use NeuronAI\Evaluation\BaseEvaluator;
use NeuronAI\Evaluation\Contracts\DatasetInterface;
use NeuronAI\Evaluation\Conversation\Conversation;
use NeuronAI\Evaluation\Dataset\JsonDataset;
use NeuronAI\Evaluation\Trajectory\Trajectory;
use NeuronAI\Workflow\Interrupt\ApprovalRequest;

class RefundConversationEvaluator extends BaseEvaluator
{
    private AgentInterface $judge;

    public function setUp(): void
    {
        $this->judge = JudgeAgent::make();
    }

    public function getDataset(): DatasetInterface
    {
        return new JsonDataset(__DIR__ . '/datasets/refunds.json');
    }

    public function run(array $datasetItem): mixed
    {
        return Conversation::make(RefundAgent::make())
            ->withTurns($datasetItem['turns'])
            ->withApprovals(function (ApprovalRequest $request, Trajectory $soFar) use ($datasetItem): array {
                $payload = [];

                foreach ($request->getActions() as $action) {
                    $payload[$action->id] = $datasetItem['decisions'][$action->name] ?? 'approve';
                }

                return $payload;
            })
            ->run();
    }

    public function evaluate(mixed $trajectory, array $datasetItem): void
    {
        // The agent followed the essential path
        $this->assert(new TrajectoryMatches($datasetItem['expected_tools'], Mode::Subset), $trajectory);

        // The refund was denied by the approver
        $this->assert(new ToolWasRejected('refund_order'), $trajectory);

        // ...and the agent communicated it instead of pretending it worked
        $this->assert(new StringContains('cannot'), $trajectory->finalAnswer());

        // Overall: did the conversation serve the user, given the denial?
        $this->assert(new TaskCompletionJudge($this->judge, goal: $datasetItem['goal']), $trajectory);
    }
}
```

With the matching dataset:

```json
[
    {
        "goal": "Get a refund for order #123",
        "turns": [
            "Hi, I want a refund for order #123",
            "Yes, please proceed."
        ],
        "decisions": {
            "refund_order": ["reject", "Amount exceeds the automatic refund limit"]
        },
        "expected_tools": ["search_orders", "refund_order"]
    }
]
```

Everything else you know about the evaluation framework applies unchanged: the Trajectory is just the output of `run()`, so parallel execution, output drivers, and the CLI runner all work exactly as before.
