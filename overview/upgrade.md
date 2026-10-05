---
metaLinks:
  alternates:
    - https://app.gitbook.com/s/GHx4l2LknIex7vFIUg1R/overview/upgrade
---

# Upgrade

{% hint style="warning" %}
### Agentic Upgrade (recommended)

We documented the entire upgrade process in a dedicated directory **`vendor/neuron-core/neuron-ai/upgrade`**. You can point your coding agent to this directory and it will automatically receive accurate instructions to upgrade your code. Here is the prompt you can use:

```
I updated the `neuron-core/neuron-ai` dependency from version 3.x to 4.x. 
Please look at the upgrade guide at `vendor/neuron-core/neuron-ai/upgrade` 
and update my application code if necessary.
```
{% endhint %}

## Upgrade to v4 from v3

In this new major version the public APIs of Neuron components weren't changed dramatically (we minimized the impact as much as possible). We've focused on improving the most important component on which the entire framework is built on. Workflow is the foundation of the entire architecture, the changes implemented in this new version may impact your code, especially if you use patterns like tool approval, interruption, and Streaming Adapters. We recommend to use the upgrade guides for coding agents.

We also took advantage of this release to fix other critical issues emerged in the v3 like the Tool Approval flow in the Agent, and other design improvements to have more freedom to evolve the framework with less breaking changes in the future.

We continue to work to provide the best possible developer experience, to help you create successful AI products in PHP.

## Updating Dependencies

You should update the following dependencies in your application's `composer.json` file:

```json
{
    "require": {
        ...,
        "neuron-core/neuron-ai": "^4.0",
    },
}
```

The `inspector-php` package was removed from default dependencies. So you have to install it in your application if you want to connect your agent to the [Inspector](https://inspector.dev/) monitoring dashboard:

```shellscript
composer require inspector-apm/inspector-php
```

## High Impact Changes

### New Agentic Skills

Skills for coding agents have been completely rewritten and reorganized to reflect the improvements and new features of this new major version. We recommend to remove the skills directory inside your coding agent folder (e.g. `.claude`, `.agent`) and follow the installation guide again.

<a href="agentic-development.md" class="button primary" data-icon="arrow-right-long">Agentic Skills</a>

### Agent return type

In this new major version the Agent entity was subject to a substantial refactor in order to eliminate many frictions for advanced use cases, and make the agent class more usable as a normal Workflow from which it inherits.

The most important impact is the return type. The Agent return an instance of the `AgentState` that is an extension of the underlying `WorkflowState` with a couple of helper methods to keep as much as possible the external APIs seen by your application unchanged.

The most impactful change is in the `stream()` method. Without the `AgentHandler` the method returns the generator directly.

```php
foreach ($agent->stream(new UserMessage("Hello")) as $event) {
    if ($event instanceof TextChunk) {
        echo $event->content;
    }
}
```

If you interact with frontend protocol, you must register the stream adapter on the agent instance.

```php
$generator = MyAgent::make()
    ->setStreamAdapter(new AGUIAdapter('thread_id'))
    ->stream(
        new UserMessage("Hello")
    );

foreach ($generator as $event) {
    echo $event;
    ob_flush();
    flush();
}
```

nothing change for `structured()` and `chat()`.

### Tool becomes fully abstract

The `Tool` class is now abstract and can no longer be used directly. Its design is now intended to be extendable, allowing you to implement your own tools with less code and more flexibility.

We also removed the constructor from the abstract class so you can specify tool name and description as normal class properties instead of calling the parent constructor. You are free to use a class constructor only if you want to pass external dependencies to the tool:

```php
class MyTool extends Tool
{
    protected string $name = 'my_tool';
    
    protected ?string $description = 'What the tool does.';
    
    public function __construct(protected string $apiKey){}
    
    public function __invoke()
    {
        ...
    }
}
```

### Remove WorkflowHandler

The Workflow component was subject of an important refactoring in order to simplify its usage and public APIs. Working with Workflow in the previous version, you were need to call the `init()` method to get the `WorkflowHandler` instance and than call `run()` or `events()` on the handler to finally execute the workflow:

```php
$handler = MyWorkflow::make()->init();

// One shot run
$finalState = $handler->run();

// Stream events
foreach($handler->events() as $chunk) {
    // ...
}
$finalState = $handler->getResult();
```

Following a drastic simplification of the workflow execution logic, the handler is no longer necessary and it is possible to invoke the two methods `run()` and `events()` directly on the workflow.

```php
// One shot run
$finalState = MyWorkflow::make()->run();

// Stream events
$generator = MyWorkflow::make()->events();
foreach($generator as $chunk) {
    // ...
}
$finalState = $generator->getResult();
```

### Workflow Interrupt/Resume

The architecture of the workflow execution and its interruption capabilities was redisigned to make it easier to manage interruption and tool approval, but also open the doors for the implementation of durable, crash proof, agentic workflows.

#### Remove WorkflowInterrupt exception

In case of interruption the Workflow doesn't throw the special `WorkflowException` to inform the caller script about the interruption. It just returns an "interrupted" state:

```php
$state = $workflow->run();

if ($state->isInterrupted()) {
    // Use the information in the request
    $request = $state->getInterruptRequest();
    // The resume token is auto-generated and available from the workflow instance
    $workflowId = $workflow->getWorkflowId();
}
```

No more try/catch block.

This change the [Tool Approval](../agent/tool-approval.md) flow. Check out the documentation in case you are using this middleware in your agents.

#### Resume Payload as plain array

The interruption request you propagate from the node is now only a signal to carry information from the node to the outside caller script. To resume the workflow you no longer need to pass the request back to the workflow. The resume payload is now just a simple array:

```php
// Example of a node calling interrupt()
class InterruptableNode extends Node
{
    public function __invoke(FirstEvent $event, WorkflowState $state): NextEvent
    {
        $payload = $this->interrupt(new ApprovalRequest('human input needed'));
        $state->set('received_feedback', $payload);
        return new NextEvent();
    }
}

// Define the inbound payload — it will be returned by the interrupt() method
$payload = ['action_id' => 'approve'];

$finalState = $workflow->resume($payload);
```

### Agent Instructions

Agent instructions must be an instance of the new message type `SystemMessage`. You can just pass the string to the constructor to make it compatible with this new version:

```php
use NeuronAI\Chat\Messages\SystemMessage;

class YouTubeAgent extends Agent
{
    protected function provider(): AIProviderInterface
    {
        ...
    }
    
    protected function instructions(): SystemMessage
    {
        return new SystemMessage(
            "You are an AI Agent specialized in writing YouTube video summaries"
        );
    }
}
```

### Tool Approval

The tool approval flow was entirely rewritten. The Agent class now manages the entire process. You just need to take care of rendering the UI so users can decide whether to approve or reject a tool call.

The status of tools requiring approval is stored into the chat history within the last `ToolCallMessage`. This allows you to design the UI to just render the messages in the chat history, and when it meets a `ToolCallMessage` you can check the approval status of the tools to show the Approve/Deny actions, or the normal tool call already happened.

<a href="../agent/tool-approval.md" class="button primary" data-icon="arrow-right-long">Tool Approval</a>

#### New Tool `requiresApproval()` method

We introduced the `requiresApproval()` method on the Tool class to determine whether approval is needed based on the tool call's arguments:

```php
class MyTool extends Tool
{
    ...,
    
    public function requiresApproval(array $inputs): bool
    {
        return $inputs['amount'] > 100;
    }
}
```

To activate the tool approval flow you always need to attach the [ToolApproval](../agent/tools.md#tool-approval) middleware to the Agent. Custom approval policy on the middleware have precedence over the one defined in the tool's `requiresApproval()` method.

### Database Schema For Workflow Persistence

Due to the changes in the workflow execution model, the database schema for persistence across interruptions has been changed to support the new features. Check out the dedicated section to get ready to run SQL queries to start with the new database format.

<a href="../workflow/persistence.md" class="button primary" data-icon="arrow-right-long">Persistence</a>

### Guzzle dependency removed

In the previous version guzzle was a required composer dependency to power the framework default `GuzzleHttpClient`. This new major version ships with the default `CurlHttpClient` that does not require any package dependency, just `ext-curl` that should be already available in any PHP installation.

If you are passing a custom GuzzleHttpClient instance to framework components you have to explicitly require guzzle in your application.

```shellscript
composer require guzzlehttp/guzzle
```

### Built-In Vector Store Filtering

`VectorStoreInterface` changed to support built-in filtering capabilities. Methods changed their name and signature. If you are implementing `VectorStoreInterface` by yourself you should migrate your implementation to the new contract.

<a href="../rag/vector-store.md" class="button primary" data-icon="arrow-right-long">Vector Stores</a>

### Calculator Toolkit

It turned out that models struggled a lot in composing multiple tool calls to resolve mathematical expressions, and instead they are really good in expression representation. The CalculatorToolkit was refactored with a main `Evaluate` tool, removing tool representing single math operations: `add`, `subtract`, `multiply`, etc. The model can now describe the formula it wants to solve and the evaluate tool will interpret this expression and return the result, all in one turn, no matter how complex the expression is. This strategy also saves a lot of tokens, since less tools are sent to the provider API.

<a href="../agent/tools.md#toolkits" class="button primary" data-icon="arrow-right-long">Calculator Toolkit</a>

### Chat History

Chat history was subject of major refactor to achieve two goals:

* Separate the message store from the history and context window management
* Making Neuron integration in your application easier

`ChatHistoryInterface` was removed. The new public APIs are backed by the new `MessageStoreInterface`. As the name says, the message store is responsible only for storing your messages in a persistence layer. It marks messages that fall out of the context window as `archived` instead of deleting them, so the model sees a trimmed thread while your storage keeps the full history.

{% hint style="warning" %}
We recommend to rely on the agentic upgrade process to move your Agent and history implementation to the new APIs.
{% endhint %}

<a href="../agent/chat-history-and-memory.md" class="button primary" data-icon="arrow-right-long">Chat History</a>

### Middleware Signature

The Workflow engine allows you to declare `resources` you want to carry during execution that will not need to be saved during interruptions. Middleware receive resources too, so you can interact with these items during workflow execution. In an Agent for example, resources contains tools, agent instructions, and the chat history. Middleware methods now get an additional argument `$resources`.

```php
interface WorkflowMiddleware
{
    public function before(
        NodeInterface $node, 
        Event $event, 
        WorkflowState $state, 
        WorkflowResources $resources, // Middleware receive resources next to the state.
    ): void;

    public function after(
        NodeInterface $node, 
        Event $result, 
        WorkflowState $state, 
        WorkflowResources $resources, // Middleware receive resources next to the state.
    ): void;
}
```

### Change ReaderInterface contract

The contract had a single static method `getText()`. The static method makes real readers instances with custom constructions meaningless. The interface now enforce the implementation of concrete instances instead of static ones:

{% hint style="warning" %}
**Custom readers must be adjusted accordingly**. The agentic upgrade will cover this refactor.
{% endhint %}

```php
interface ReaderInterface
{
    public function read(string $filePath): string;
}
```

## Medium Impact

### Monitoring

The framework is transitioning to the [PSR-14 Event Dispatcher](https://www.php-fig.org/psr/psr-14/) interface, instead of the PHP native \SplObserver. We kept the existing interfaces in place, and also the `LogObserver` using adapters, but they are marked as `@deprecated`.

This will make it easier to integrate Neuron agents and agentic workflows in general with other existing frameworks and applications.

<a href="../agent/observability.md" class="button primary" data-icon="arrow-right-long">Monitoring</a>

### Providers return ProviderResponse

AI provider methods `chat()` and `stream()` now return `ProviderResponse` instead of `Message`. The `ProviderResponse` wraps the assistant message and provides access to the raw HTTP response body and headers.

**This only affects standalone provider usage.** When providers are used inside an Agent (via `chat()`, `stream()`, or `structured()` on the Agent itself), no changes are needed — the Agent handles the `ProviderResponse` internally.

You only need to refactor code that calls provider methods directly, such as in scripts, controllers, commands, or custom workflows.

```php
$response = $provider->chat(new UserMessage(...));

// Get the assistance Message
$response->message();

// Get the raw provider body
$response->body();

// Get the headers
$response->headers();
```

### Changes on AIProviderInterface

We removed `messageMapper()` and `toolPayloadMapper()` from the `AIProviderInterface`, and the new `getModel()` methos was introduced. If you have any custom provider implementation that directly use this interface, you need to adjust it properly.

```php
interface AIProviderInterface
{
    public function getModel(): string;

    public function systemPrompt(string|array|null $prompt): AIProviderInterface;

    public function setTools(array $tools): AIProviderInterface;

    public function chat(Message ...$messages): ProviderResponse;

    public function stream(Message ...$messages): Generator;

    public function structured(array|Message $messages, string $class, array $response_schema): ProviderResponse;

    public function setHttpClient(HttpClientInterface $client): AIProviderInterface;
}
```

## New Features

### AG-UI Custom Events

AG-UI uses its native `STEP_STARTED`, `STEP_FINISHED`, `ACTIVITY_SNAPSHOT`, and `CUSTOM` events. Vercel emits transient `data-*` parts, so intermediate domain information is available to the UI without being added to assistant-message history.

You can map application events by its exact class when domain code should remain independent from Neuron's portable event objects.

<a href="../agent/streaming.md#custom-events" class="button primary" data-icon="arrow-right-long">UI Protocol Custom Events</a>

### Streaming Channels

As agents become more interactive and capable, application developers are increasingly forced to run them outside the HTTP request lifecycle because of its timeout limits, which leaves the streamed output with no way to reach the UI. Channels deliver the streamed output of Agents and Workflows to the user interface through external streaming systems such as [Pusher](https://pusher.com/), websockets, a Redis queue, or whatever your application already uses.

<a href="../agent/streaming.md#delivery-channels" class="button primary" data-icon="arrow-right-long">Streaming Channels</a>

### Partial Event Streaming

Realtime servers like Pusher, Socketi, or any Pusher compatible server usually put a limit on the size of the payload. Pusher caps an event at 10 KB; Reverb defaults to the same, Soketi to 100 KB.

To support streaming when your real-time server has such a limit, an event that does not fit the max size - typically a tool result, or a message snapshot - is split into consecutive fragments. Furthermore servers do not guarantee that fragments arrive in order and never interleave. So the browser must be able to reconcile this fragmentation and ordering in order to deliver consistent streaming to your UI components.

Neuron 4 ships with a first-party Typescript module that brings this capability in your frontend.

<a href="../agent/streaming.md#event-fragmentation" class="button primary" data-icon="npm">@neuron-core/streaming</a>

### Semantic Memory

Your agent can now remember what matters to a person across multiple conversations. Users with multiple active threads can see the agent remember their past conversations and preferences.

V4 introduces the `SemanticMemoryRetrieval` component, which automatically stores and recalls memories across user sessions. It belongs to the RAG agent can be customized or replaced to fit your application logic.

<a href="../rag/retrieval.md#semantic-memory" class="button primary" data-icon="arrow-right-long">Semantic Memory</a>

### StoppableHttpClient

Stop a streamed answer mid-flight, for a "stop generating" button. The provider keeps the text streamed so far as a partial content and store it in the chat history.

<a href="../agent/async.md#stoppablehttpclient" class="button primary" data-icon="arrow-right-long">Http Client</a>

### Classifier

In Neuron AI you could already answer them with [structured output](https://docs.neuron-ai.dev/agent/structured-output): describe the allowed answers, force the response into a PHP class, read the property. It is the same mechanism behind the [AI as a judge](https://docs.neuron-ai.dev/agent/evaluation#ai-as-a-judge) pattern in agent evaluations. It works well when the decision is taken once in a while. The trouble begins when you want it on every turn of every conversation, because a generative model produces text one token after the other, and you are paying a general purpose writer, with its latency and its price, to obtain one word. A guardrail that doubles the response time of the agent gets switched off at the first complaint. A judge that costs as much as the agent it evaluates runs on a sample of the traffic, if it runs at all.

A classifier has no conversation and no text to stream. The Neuron AI Classifier has its own contract, `ClassifierInterface`, for asking closed questions about some input and receiving probabilities back. You can use it to take decisions inside your Agents or Workflows like guardrails, prompt injections, score the quality of a response, etc.

<a href="../providers/classifier.md" class="button primary" data-icon="arrow-right-long">Classifier</a>
