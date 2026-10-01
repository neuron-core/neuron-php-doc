---
description: Stream real -time updates during workflow execution
metaLinks:
  alternates:
    - https://app.gitbook.com/s/GHx4l2LknIex7vFIUg1R/workflow/streaming
---

# Streaming

Workflows can be complex, they are designed to handle complex, branching, itarable logic, which means they can take time to fully execute. To provide your user with a good experience, you may want to provide an indication of progress by streaming events as they occur. Workflows have built-in support for this directly from inside the nodes.

### Emit events from nodes

Let's set up a new event to handle streaming our progress as we go:

```php
namespace App\Neuron;

class ProgressEvent implements Event 
{
    public function __construct(protected string $msg){}
}
```

We'll take our example MyWorkflow with multiple nodes from the previous tutorial and modify the nodes  to stream upadtes instead of echoing output directly.

{% hint style="warning" %}
**Notice**: To stream events from node you need to add `\Generator` as additional return type of the `__invoke` method.
{% endhint %}

```php
namespace App\Neuron;

use Generator;
use NeuronAI\Workflow\Node;
use NeuronAI\Workflow\StartEvent;
use NeuronAI\Workflow\StopEvent;

class InitialNode extends Node
{
    public function __invoke(
        StartEvent $event, 
        WorkflowState $state,
        WorkflowResources $resources
    ): Generator|FirstEvent {
        yield new ProgressEvent("Handling StartEvent");
        
        return new FirstEvent("InitialNode complete");
    }
}

class NodeOne extends Node
{
    public function __invoke(
        FirstEvent $event, 
        WorkflowState $state,
        WorkflowResources $resources
    ): Generator|SecondEvent {
        yield new ProgressEvent($event->firstMsg);
        
        return new SecondEvent("NodeOne complete");
    }
}

class NodeTwo extends Node
{
    public function __invoke(
        SecondEvent $event, 
        WorkflowState $state,
        WorkflowResources $resources
    ): Generator|StopEvent {
        yield new ProgressEvent($event->secondMsg);
        
        yield new ProgressEvent("NodeTwo complete");
        
        $state->set('message', 'Streaming end');
        
        return new StopEvent();
    }
}
```

To actually get this output, we need to start the workflow and listen for the events, like this:

```php
$workflow = Workflow::make()
    ->setWorkflowId('wk_id')
    ->addNodes([
        new InitialNode(),
        new NodeOne(),
        new NodeTwo(),
    ]);

$stream = $workflow->events();

foreach ($stream as $event) {
    if ($event instanceof ProgressEvent) {
        echo "\n- ".$event->message;
    }
}

$finalState = $stream->getResult();

// It will print "Streaming end"
echo "\n- ".$finalState->get('message');
```

The full output will be:

```
- Handling StartEvent
- InitialNode complete
- NodeOne complete
- NodeTwo complete
- Streaming end
```

### Stream Agent Output

Running Agents inside nodes is one of the most common use case working with workflow. You may be interested in directly stream an internal agent output to the client to give real time feedback of the underlying generation. You can do it by simply streaming the agent's output from within the node.

```php
class InitialNode extends Node
{
    public function __invoke(
        StartEvent $event, 
        WorkflowState $state,
        WorkflowResources $resources
    ): Generator|FirstEvent {
        // Run an agent with streaming
        yield from Agent::make()
            ->stream(new UserMessage($state->get('prompt')));
        
        return new FirstEvent("InitialNode complete");
    }
}
```

To get this output you can listen for workflow events as usual:

```php
$workflow = MyWorkflow::make()->setWorkflowId('wk_id');

foreach ($workflow->events() as $event) {
    echo match($event::class) {
        TextChunk::class => "\n- ".$event->content,
        ...
    }
}
```

### Monitoring & Debugging

Many of the applications you build with Neuron will contain multiple steps with multiple invocations of LLM calls. As these applications get more and more complex, it becomes crucial to be able to inspect what exactly is going on inside your agentic system. The best way to do this is with [Inspector](https://inspector.dev/).

{% embed url="https://docs.inspector.dev/guides/neuron-ai" %}
