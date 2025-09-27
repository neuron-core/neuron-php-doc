---
description: The key breakthrough is that interruption isn't a bug – it's a feature.
---

# Human In The Loop

Neuron Workflow supports a robust **human-in-the-loop** pattern, enabling human intervention at any point in an automated process. This is especially useful in large language model (LLM)-driven applications where model output may require validation, correction, or additional context to complete the task.

Here's how it works technically:

**Interruption Points**: Any node in your Workflow can request an interruption by specifying the data it want to present to the human. This could be a simple yes/no decision, a content review, data validation, or structured data.

**State Preservation**: When an interruption happens, Neuron automatically saves the complete state of your Workflow. Your Workflow essentially goes to sleep, waiting for human input.

**Resume Capability**: Once a human provides the requested input, the Workflow wakes up exactly from the node it left off. No data is lost, no context is forgotten.

**External Feedback Integration**: The human input is injected into the interrupted node to be consumed on resume.

### Interruption

When a Neuron Workflow encounters an interruption, it doesn't simply stop—it preserves its entire state, and waits for guidance before proceeding. This allows oyu to creates a hybrid intelligence system where AI handles the computational heavy lifting while humans contribute to strategic oversight, domain expertise, and decision-making.

The simplest way to can ask for an interruption is calling the `interrupt()` method inside a node:

```php
<?php

namespace App\Neuron;

use NeuronAI\Workflow\Node;
use NeuronAI\Workflow\WorkflowState;

class InterruptionNode extends Node
{
    public function __invoke(InputEvent $event, WorkflowState $state): OutputEvent
    {
        // Interrupt the workflow and wait for the feedback.
        $feedback = $this->interrupt([
            'question' => 'Should we continue?',
            'current_value' => $state->get('accuracy')
        ]);
    
        if ($feedback['approved']) {
            $state->set('is_sufficient', true);
            $state->set('user_response', $feedback['response']);
            return new OutputEvent();
        }
        
        $state->set('is_sufficient', false);
        return new InputEvent();
    }
}
```

Calling the `interrupt()` method you can pass the information you need to interact with the human. You will be able to catch this data later, outside of the workflow so you can inform the user with relevant information from inside the Workflow to ask for feedback.&#x20;

When the Workflow will be awakened it will restart from this node, and the `$feedback` variable will receive the human's response data.

### Checkpointing

When the Workflow is awakened it restarts the execution from the node where it was interrupted. The node will be re-executed entirely including the code present before the interruption.

If you need to call for an interruption not at the beginning of the node, but after performing other operations, you can use checkpoints to save the result of some statements to be used when the node is re-starded. Here is an example:

```php
<?php

namespace App\Neuron;

use NeuronAI\Workflow\Node;
use NeuronAI\Workflow\WorkflowState;

class InterruptionNode extends Node
{
    public function __invoke(InputEvent $event, WorkflowState $state): OutputEvent
    {
        // The result of this code block is saved and returned when the workflow is awakened.
        $sentiment = $this->checkpoint('agent-1', function () {
            return MyAgent::make()->structured(
                new UserMessage(...),
                SentimentResult::class
            );
        });
        
        // Interrupt the workflow and wait for the feedback.
        if ($sentiment->isNegative()) {
            $feedback = $this->interrupt([
                'question' => 'Should we continue?',
                'current_value' => $state->get('accuracy')
            ]);
        }
    
        if ($feedback['approved']) {
            $state->set('is_sufficient', true);
            $state->set('user_response', $feedback['response']);
            return new OutputEvent();
        }
        
        $state->set('is_sufficient', false);
        return new InputEvent();
    }
}
```

### Consume The Feedback Directly

You can also consume the external feedback somewhere in your code other than where you call the `interrupt()` method.

The `consumeInterruptFeedabck()` method allows you get the value of the external feedback or null if the node is simply running and not awakening:

```php
<?php

namespace App\Neuron;

use NeuronAI\Workflow\Node;
use NeuronAI\Workflow\WorkflowState;

class InterruptionNode extends Node
{
    public function __invoke(InputEvent $event, WorkflowState $state): OutputEvent
    {
        // Interrupt the workflow and wait for the feedback.
        $feedback = $this->consumeInterruptFeedback();
    
        if ($feedback['approved'] ?? false) {
            $state->set('is_sufficient', true);
            $state->set('user_response', $feedback['response']);
            return new OutputEvent();
        }
        
        $this->interrupt([
            'question' => 'Should we continue?',
            'current_value' => $state->get('accuracy')
        ]);
        
        $state->set('is_sufficient', false);
        return new InputEvent();
    }
}
```

This allows much more flexibility if you need to condition the beginning of the node based on the given feedback.

### Conditional Interruption

You can also use `interruptIf()` as an helper to evaluate a conditional interruption:

```php
<?php

namespace App\Neuron;

use NeuronAI\Workflow\Node;
use NeuronAI\Workflow\WorkflowState;

class InterruptionNode extends Node
{
    public function __invoke(InputEvent $event, WorkflowState $state): OutputEvent
    {
        // Conditional interruption
        $this->interruptIf(
            $state->get('is_sufficient', false), 
            ['question' => 'Should we continue?']
        );
        
        // Using a callback to evaluate the condition
        $this->interruptIf(
            fn() => $state->get('is_sufficient', false), 
            ['question' => 'Should we continue?']
        );
        
        return new InputEvent();
    }
}
```

### Catch the Interruption

To be able to interrupt and wake up a Workflow you need to provide a persistence layer and a workflow ID when creating the Workflow instance:

```php
$workflow = new WorkflowAgent(
    new FilePersistence(__DIR__),
    'CUSTOM_ID'
);
```

The `ID` is the reference to save and load the state of a specific Workflow during the interruption and wake up process. When a node call for an interruption it fires a special type of exception represented by the `WorkflowInterrupt` class. You can catch this exception to manage the interruption request.

```php
try {
    $result = $workflow->start()->getResult();
} catch (WorkflowInterrupt $interrupt) {
    $data = $interrupt->getData();
    
    /*
     * Store $data['question'], $data['current_value'] and the Workflow-ID,
     * and alert the user to provide a feedback.
     */
}
```

Use the information in the `$data` array to guide the human in providing a feedback. Once you finally have the user's feedback you can resume the workflow. Remeber to use the same `ID` of the interrupted execution.

```php
$workflow = new WorkflowAgent(
    new FilePersistence(__DIR__),
    'CUSTOM_ID' // <- Use the same ID of the interrupted workflow
);

// Resume the Workflow passing the human feedback
$result = $workflow-wakeup(['approved' => true])->getResult();

// Get the final answer
echo $result->get('answer');
```

You can take a look at the script below as an example of this process:&#x20;

{% @github-files/github-code-block url="https://github.com/inspector-apm/neuron-ai/blob/main/examples/workflow/workflow-interrupt.php" %}

### Monitoring & Debugging

Before moving into the Workflow creation process, we recommend having the monitoring system in place. It could make the learning curve of how Workflow works much more easier. The best way to monitoring Workflow is with [Inspector](https://inspector.dev/).

After you sign up at the link above, make sure to set the `INSPECTOR_INGESTION_KEY` variable in the application environment file to monitoring Workflow execution:

{% code title=".env" %}
```
INSPECTOR_INGESTION_KEY=nwse877auxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```
{% endcode %}
