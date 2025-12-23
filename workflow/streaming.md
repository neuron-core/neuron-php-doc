---
description: Stream real -time updates during workflow execution
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

We'll take our example MyWorkflow with multiple nodes from the previous tutorial and modify the nodes  to stream updates instead of echoing output directly:

```php
namespace App\Neuron;

use NeuronAI\Workflow\Node;
use NeuronAI\Workflow\StartEvent;
use NeuronAI\Workflow\StopEvent;

class InitialNode extends Node
{
    public function __invoke(StartEvent $event, WorkflowState $state): \Generator|FirstEvent
    {
        yield new ProgressEvent("Handling StartEvent");
        
        return new FirstEvent("InitialNode complete");
    }
}

class NodeOne extends Node
{
    public function __invoke(FirstEvent $event, WorkflowState $state): \Generator|SecondEvent
    {
        yield new ProgressEvent($event->firstMsg);
        
        return new SecondEvent("NodeOne complete");
    }
}

class NodeTwo extends Node
{
    public function __invoke(SecondEvent$event, WorkflowState $state): \Generator|StopEvent
    {
        yield new ProgressEvent($event->secondMsg);
        
        yield new ProgressEvent("NodeTwo complete");
        
        $state->set('message', 'Streaming end');
        
        return new StopEvent();
    }
}
```

{% hint style="warning" %}
To stream events from node you need to add `\Generator` as additional return type of the `__invoke` method.
{% endhint %}

To actually get this output, we need to start the workflow and listen for the events, like this:

```php
$handler = Workflow::make()
    ->addNodes([
        new InitialNode(),
        new NodeOne(),
        new NodeTwo(),
    ])
    ->start();

foreach ($handler->streamEvents() as $event) {
    if ($event instanceof ProgressEvent) {
        echo "\n- ".$event->message;
    }
}

$finalState = $handler->getResult();

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

Running Agents inside nodes is one of the most common use case working with workflow. You may be interested in directly stream the agent output to the client to give real time feedback of the underlying generation. Workflow allows you to do this by simply streaming the agent's output from within the node.

```php
class InitialNode extends Node
{
    public function __invoke(StartEvent $event, WorkflowState $state): \Generator|FirstEvent
    {
        // Run an agent with streaming
        $stream = Agent::make()->stream(new UserMessage($state->get('prompt')));

        foreach ($stream as $text) {
            yield new GenerationProgressEvent($text);
        }
        
        return new FirstEvent("InitialNode complete");
    }
}
```

To get this output you can listen for workflow events as usual:

```php
$handler = Workflow::make()
    ->addNodes([
        new InitialNode(),
        new NodeOne(),
        new NodeTwo(),
    ])
    ->start();

foreach ($handler->streamEvents() as $event) {
    echo match(getClass($event)) {
        ProgressEvent::class => "\n- ".$event->message,
        GenerationProgressEvent::class => $event->text,
        default => ''
    }
}
```

### Monitoring & Debugging

Before moving into the Workflow creation process, we recommend having the monitoring system in place. It could make the learning curve of how Workflow works much more easier. The best way to monitoring Workflow is with [Inspector](https://inspector.dev/).

After you sign up at the link above, make sure to set the `INSPECTOR_INGESTION_KEY` variable in the application environment file to monitoring Workflow execution:

{% code title=".env" %}
```
INSPECTOR_INGESTION_KEY=nwse877auxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```
{% endcode %}
