---
description: The key breakthrough is that interruption isn't a bug – it's a feature.
---

# Interruption

### What it is

Neuron's interruption pattern provides a built-in _human-in-the-loop_ mechanism that allows\
workflows to pause execution and wait for external input before resuming.&#x20;

At its core,&#x20;interruptions are implemented through the abstract `InterruptRequest` class, a framework primitive that developers can extend to create custom interruption experiences tailored to their\
application's specific needs. The framework includes an `ApprovalRequest` as a built-in implementation covering the most common use case of approving actions (such as tool calls), but the architecture is intentionally flexible: any workflow node or middleware can trigger an interruption, and the persistence layer ensures state is preserved across the pause/resume cycle, making it suitable for long-running processes that require human decision points at any stage.

If the built-in `ApprovalRequest` doesn't fit with your use case, you are free to create your custom interrutpion request to create a specific UI experience.

Here's how it works:

**Interruption Points**: Any node in your Workflow can request an interruption by specifying the data it want to present to the human. This could be a simple yes/no decision, an alert, or any structured data.

**State Preservation**: When an interruption happens, Neuron automatically saves the complete state of your Workflow. Your Workflow essentially goes to sleep, waiting for human input.

**Resume**: Once a human responde to the interruption request, the Workflow wakes up exactly from the node it left off. No data is lost, no context is forgotten.

**External Feedback Integration**: The edited interruption request is injected into the interrupted node to be continue its execution receiving the human feedback.

### Video Introduction

We know that Interruption flow is a quite advanced feature. Even with all the documentation below it may not be easy to grasp all aspects of this architecture. We're happy to link you below to an introductory video made by our community member [Amitav Roy](https://www.linkedin.com/in/royamitav/).

It might give you some additional information that, combined with the documentation, can help you understand how to implement your use cases.

{% embed url="https://www.youtube.com/watch?v=jjEBjTRDLZE" %}

### How it works

When you call for an interruption, the Workflow doesn't simply stop, it preserves its entire state, and waits for guidance before proceeding. This allows you to creates a hybrid intelligence system where AI handles the computational heavy lifting while humans contribute to strategic oversight, and decision-making.

The simplest way to ask for an interruption is calling the `interrupt()` method inside a node, providing an interruption request. Here is an example using the built-in `ApprovalRequest`:

```php
<?php

namespace App\Neuron;

use NeuronAI\Workflow\Events\Event;
use NeuronAI\Workflow\Interrupt\Action;
use NeuronAI\Workflow\Interrupt\ApprovalRequest;
use NeuronAI\Workflow\Node;
use NeuronAI\Workflow\WorkflowState;

class InterruptionNode extends Node
{
    public function __invoke(InputEvent $event, WorkflowState $state): OutputEvent
    {
        // Interrupt the workflow and wait for the feedback.
        $humanResponse = $this->interrupt(
            new ApprovalRequest(
                message: 'Should I continue?'
                actions: [
                    new Action('delete_file', 'Delete File', 'Delete /var/log/old.txt'),
                ],
            )
        );
    
        $action = $humanResponse->getAction('delete_file');
    
        if ($action->isApproved()) {
            $state->set('is_sufficient', true);
            $state->set('user_feedback', $action->feedback);
            return new OutputEvent();
        }
        
        $state->set('is_sufficient', false);
        return new InputEvent();
    }
}
```

You can eventually implement your custom interruption request to pass the information you need for the human interaction. You will be able to catch this data later, outside of the workflow so you can inform the user for feedback.&#x20;

When the Workflow will be resumed it will restart from the same node it was interrupted, and the `$feedback` variable will receive the human's response data.

The `InterruptRequest` follows a **request-response pattern** where:

1. **Request Phase**: A workflow node identifies actions requiring human approval and creates an `InterruptRequest` containing details of those actions
2. **Pause Phase**: The workflow throws a `WorkflowInterrupt` exception, preserving the entire execution context
3. **Decision Phase**: The application presents actions to users, who approve, reject, or edit each action
4. **Resume Phase**: The workflow resumes with user decisions, continuing execution based on the feedback

This design ensures workflow can safely pause at any point, persist its state, and resume exactly where it left off, even across different sessions.

### Custom Interruption Request

The `InterruptRequest` is the central component of Neuron's human-in-the-loop (HITL) pattern, designed to pause workflow execution and request human approval or input for specific actions. It provides a structured, type-safe approach to building interactive AI workflows.

You can create your own implementation and feed it into the interrupt method.

```php
class ContentReviewInterrupt extends InterruptRequest
{
    public function __construct(
        protected string $message,
        protected string $content
    ) {
        parent::__construct($message)
    }
    
    public function getContent(): string
    {
        return $this->content;
    }
    
    public function jsonSerialize(): array
    {
        return [
            'message' => $this->message,
            'content' => $this->content,
        ];
    }
    
    public static function fromArray(array $data)
    {
        return new static($data['message'], $data['content']);
    }
}
```

Use it for your interrutpion use case:

```php
class InterruptionNode extends Node
{
    public function __invoke(InputEvent $event, WorkflowState $state): OutputEvent
    {
        // Generate an article
        $response = ContentCreatorAgent::make()
            ->chat(new UserMessage($event->prompt))
            ->getMessage();
    
        // Interrupt the workflow and wait for the feedback.
        $reviewRequest = $this->interrupt(
            new ContentReviewInterrupt(
                message: 'This is the new article. Review the content before saving it to the database.'
                $response->getContent()
            )
        );
        
        // Save the content of the updated interrupt request
        $state->set('content', $reviewRequest->getContent());
        
        return new InputEvent();
    }
}
```

### Catching the interruption

To be able to interrupt and resume a Workflow (also Agent and RAG) you need to provide a persistence layer and a workflow ID when creating the Workflow instance:

```php
$workflow = new WorkflowAgent(
    new FilePersistence(__DIR__)
);
```

The `WORKFLOW_EXECUTION_ID` is the reference to save and load the state of a specific Workflow in case of an interruption.

When a node call for an interruption the Workflow fires a special type of exception represented by the **`WorkflowInterrupt`** class. You can catch this exception to manage the interruption request.

```php
$workflow = new WorkflowAgent(
    new FilePersistence(__DIR__),
);

try {
    return $workflow->init()->run();
} catch (WorkflowInterrupt $interrupt) {
    $request = $interrupt->getRequest();
    $resumeToken = $interrupt->getResumeToken();
    
    /*
    * You can store the request as a json object
    * along with the resume token, and ask the user for a feedback.
    */
    $pdo->prepare("INSERT INTO interruption_requests (resume_token, request) VALUES (?, ?)");
    $pdo->execute([
        $resumeToken,
        json_encode($request),
    ]);
}
```

Use the information in the `$request` object to guide the human in providing a feedback. Once you finally have the user's feedback you can resume the workflow passing the interruption request to the `init()` method. Remeber to use the same `RESUME_TOKEN`  you got during interruption.

```php
$workflow = new WorkflowAgent(
    new FilePersistence(__DIR__),
    $resumeToken // <- Use the resume token you got during interrutpion
);

$request = ContentReviewInterrupt::fromArray($data);

// Resume the Workflow passing the processed request as the feedback
$result = $workflow->init($request)->run();

// Get the final answer
echo $result->get('content');
```

You can take a look at the script below as an example of this process:&#x20;

{% @github-files/github-code-block url="https://github.com/inspector-apm/neuron-ai/blob/main/examples/workflow/workflow-interrupt.php" %}

### Checkpointing

When the Workflow is resumed it restarts the execution from the node where it was interrupted. The node will be re-executed entirely including the code present before the interruption.

If you need to call for an interruption not at the beginning of the node, but after performing other operations, you can use checkpoints to save the result of previous statements to be used when the node is resumed. Here is an example:

```php
<?php

namespace App\Neuron;

use NeuronAI\Workflow\Node;
use NeuronAI\Workflow\WorkflowState;

class InterruptionNode extends Node
{
    public function __invoke(InputEvent $event, WorkflowState $state): OutputEvent
    {
        // The result of this code block is saved and returned when the workflow is resumed.
        $sentiment = $this->checkpoint('agent-1', function () {
            return MyAgent::make()->structured(
                new UserMessage(...),
                SentimentResult::class
            );
        });
        
        // Interrupt the workflow and wait for the feedback.
        if ($sentiment->isNegative()) {
            $feedback = $this->interrupt(
                new ApprovalRequest(
                    message: 'Should I continue?'
                    actions: [
                        new Action('review_id', 'Answer review', $sentiment->content),
                    ],
                )
            );
            
            if ($feedback->getAction('review_id')->isApproved()) {
                $state->set('is_sufficient', true);
                $state->set('user_feedback', $feedback->getAction('review_id')->feedback);
                return new OutputEvent();
            }
        }
        
        $state->set('is_sufficient', false);
        return new InputEvent();
    }
}
```

The checkpoint method accepts two arguments:

* The **name** of the checkpoint must be unique in the node;
* A **Closure** to wrap the code whose result you want to save.

When the node is executed, the checkpoint method saves the result of the Closure in case of an interruption. When the node is executed again after the interruption, it can reach the interruption point with the exact same state of the previous run to get the external feedback.

### Consume The Feedback

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
    
        if ($feedback !== null && $feedback->getAction('review_id')->isApproved()) {
            $state->set('is_sufficient', true);
            $state->set('user_feedback', $feedback->getAction('review_id')->feedback);
            return new OutputEvent();
        }
        
        $this->interrupt(
            new ApprovalRequest(
                message: 'Should I continue?'
                actions: [
                    new Action('review_id', 'Answer review', $state->get('review')),
                ],
            )
        );
        
        $state->set('is_sufficient', false);
        return new InputEvent();
    }
}
```

This allows you to apply  condition at the beginning of the node based on the given feedback.

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
            $state->get('is_sufficient') == true, 
            new ApprovalRequest(
                message: 'Should I continue?'
                actions: [
                    new Action('review_id', 'Answer review', $state->get('review')),
                ],
            )
        );
        
        // Or use a callback to evaluate the condition
        $this->interruptIf(
            fn() => $state->get('is_sufficient', false), 
            new ApprovalRequest(
                message: 'Should I continue?'
                actions: [
                    new Action('review_id', 'Answer review', $state->get('review')),
                ],
            )
        );
        
        return new InputEvent();
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
