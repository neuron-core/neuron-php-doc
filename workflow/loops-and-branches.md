# Loops & Branches

Workflow makes branching and looping logic easy to implement thanks to its event driven design. Once you understand how nodes belong to events, it's easy to start imagining how you can create loops and branching, which is just deciding which event should be returned in an "if condition" or whatever logic.

### Loops

To create a loop, simply return the entry event of a previous node as the exit event of the current node. You can also use the same entry event as the current node's exit event to loop over the current node.

Take a look at the example below. The `NodeOne` can have two events as return type, `FirstEvent` and `SecondEvent`. If the node returns FirstEvent it will cause another execution of the same node because FirstEvent is handled by itself, creating a loop.

If the node returns SecondEvent it will finally move forward the execution to another node.&#x20;

```php
class NodeOne extends Node
{
    public function __invoke(FirstEvent $event, WorkflowState $state): FirstEvent|SecondEvent
    {
        echo "\n- ".$event->firstMsg;
        
        if (rand(0, 1) === 1) {
            // Returning FirstEvent it will trigger another execution of NodeOne
            return new FirstEvent("Running a loop on NodeOne");
        }
        
        return new SecondEvent("NodeOne complete, move forward");
    }
}
```

{% hint style="warning" %}
Notice the node has now two return types for the `__invoke` method: `FirstEvent` and `SecondEvent`. You have to declare all possible return events on the method signature to let the Workflow build the execution chain.
{% endhint %}

Returning FirstEvent will trigger another execution of `NodeOne`. So the final output could be:

```php
$state = Workflow::make()
    ->addNodes([
        new InitialNode(),
        new NodeOne(),
        new NodeTwo()
    ])
    ->start()
    ->getResult();

/*
- Handling StartEvent
- InitialNode complete
- Running a loop on NodeOne
- Running a loop on NodeOne
- NodeOne complete, move forward
- NodeTwo complete
*/
```

You can create a loop from any node to any other node in the workflow by defining the appropriate input event and return events of the invoke method.&#x20;

<figure><img src="../.gitbook/assets/workflow-loop.png" alt=""><figcaption></figcaption></figure>

The `NodeOne` can even return a StartEvent to jump right to the first node of the Workflow. The event driven architecutre allows you to directly point any node in the workflow both forward and backward.

### Branches

As you've already seen, you can conditionally return different events from a node to define custom execution flows. In this section we'll see an example of a workflow that branches into two different paths.&#x20;

First let's create some custom events:

```php
namespace App\Neuron;

class BrancheA1Event implements Event 
{
    public function __construct(protected string $firstMsg){}
}

class BrancheA2Event implements Event 
{
    public function __construct(protected string $secondMsg){}
}

class BrancheB1Event implements Event 
{
    public function __construct(protected string $secondMsg){}
}

class BrancheB2Event implements Event 
{
    public function __construct(protected string $secondMsg){}
}
```

In the initial node of he workflow we decide what branched we want to go through. Remeber to always define the appropriate return types in the `__invoke` method signature:

```php
class InitialNode extends Node
{
    public function __invoke(StartEvent $event, WorkflowState $state): BrancheA1Event|BrancheB1Event
    {
        if (rand(0, 1) === 1) {
            // Returning FirstEvent it will trigger another execution of NodeOne
            return new BrancheA1Event();
        }
        
        return new BrancheB1Event();
    }
}
```

The other nodes will move forward sequencially.

```php
$state = Workflow::make()
    ->addNodes([
        new InitialNode(),
        new A1Node(),
        new A2Node(),
        new B1Node(),
        new B2Node(),
    ])
    ->start()
    ->getResult();
```

<figure><img src="../.gitbook/assets/workflow-branches.png" alt=""><figcaption></figcaption></figure>

You can of course combine branches and loops in any order to fulfill the needs of your application.&#x20;

### Monitoring & Debugging

Before moving into the Workflow creation process, we recommend having the monitoring system in place. It could make the learning curve of how Workflow works much more easier. The best way to monitoring Workflow is with [Inspector](https://inspector.dev/).

After you sign up at the link above, make sure to set the `INSPECTOR_INGESTION_KEY` variable in the application environment file to monitoring Workflow execution:

{% code title=".env" %}
```
INSPECTOR_INGESTION_KEY=nwse877auxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```
{% endcode %}
