---
description: Learn how to pass data around the workflow
---

# Managing the State

Generally speaking, the purpose of a workflow is to get an input state, make able the nodes to manipulate this state during execution, and return the state as final value. Based on this architecture the state has two main roles that we will see in detail in this guide.

### Workflow Input/Output

The final return value of the workflow itself is an instance of the workflow state. So, if you need to collect the result of the workflow execution, nodes must be able to write and read from the state until the workflow ends and return the final state to the parent script.

You can also provide an initial state to workflow to feed in input values.

```php
// 1. Provide an initial state as workflow input to feed in some data
$workflow = Workflow::make(new WorkflowState(['query' => 'Hi!']))
        ->addNode(new InitialNode())
        ->addNode(...)
        ->addNode(...);

// 2. Execute the workflow and get the final state
$finalState = $workflow->start()->getReturn();

// 3. Use the final state data
echo $finalState->get('message');
```

### Using state in nodes

In our examples so far, we have passed data from node to node using properties of custom events. This is a powerful way to pass data around, but it has limitations. For example, if you want to pass data between steps that are not directly connected, you need to pass the data through all the nodes in between. This can make your code harder to read and maintain.

For this reasons we have the `WorkflowState` object available to every node in the workflow. To use it, the workflow inject the WorkflowState instance as the second argument of the node.&#x20;

```php
namespace App\Neuron;

use NeuronAI\Workflow\Node;
use NeuronAI\Workflow\StartEvent;
use NeuronAI\Workflow\StopEvent;
use NeuronAI\Workflow\WorkflowState;

class InitialNode extends Node
{
    public function __invoke(StartEvent $event, WorkflowState $state): StopEvent
    {
        $state->set('message', 'Hello World!');
        
        return new StopEvent();
    }
}

// Execute the workflow and get the final state
$finalState = Workflow::make()
    ->addNode(new InitialNode())
    ->start()
    ->getReturn();
    
// It will print "Hello World!"
echo $finalState->get('message');
```
