---
description: The key breakthrough is that interruption isn't a bug, it's a feature.
metaLinks:
  alternates:
    - https://app.gitbook.com/s/GHx4l2LknIex7vFIUg1R/workflow/human-in-the-loop
---

# Interruption

Neuron's interruption pattern lets a Workflow pause execution and wait for external input — a human decision, or an event from another system — and resume later, even in a different process, hours or days after.

In this major version the mental model rests on one rule:

> **A pause sends data&#x20;**_**out**_**&#x20;of a node. A resume brings data&#x20;**_**back in**_**.**

* **Outbound — `InterruptRequest`**: the description of the pause. A node constructs it to tell the outside world _what it is waiting for_ (actions to approve, an event name, content to review). It is immutable, and it is never handed back into the workflow.
* **Inbound — the payload array**: the answer that satisfies the pause. A plain, serialization-safe `array` delivered via `resume(payload: [...])`. The interrupted node receives it as the **return value** of the suspend call.

### How it works

An AI workflow often needs to stop mid-flight, typically to ask a human for approval before acting, without keeping a PHP process alive while the human decides. To pause, a node calls `interrupt()`: the workflow stops traversal, persists its progress, and returns normally to the caller with the state marked as interrupted.

```php
namespace App\Neuron;

use NeuronAI\Workflow\Events\Event;
use NeuronAI\Workflow\Interrupt\Action;
use NeuronAI\Workflow\Interrupt\ApprovalRequest;
use NeuronAI\Workflow\Node;
use NeuronAI\Workflow\WorkflowState;

class ApprovalNode extends Node
{
    public function __invoke(
        PurchaseEvent $event, 
        WorkflowState $state,
        WorkflowResources $resources
    ): Event {
        // Suspend the workflow, carrying the request OUTBOUND.
        $payload = $this->interrupt(
            new ApprovalRequest(
                message: 'Do you approve this purchase?',
                actions: [
                    new Action(
                        id: 'purchase_1',
                        name: 'Purchase',
                        description: "Buy {$event->item} for {$event->price}$",
                    ),
                ],
            )
        );

        // Code below this line runs ONLY on resume.
        // $payload is the INBOUND answer delivered by resume() — your node interprets it.
        if (($payload['purchase_1'] ?? null) === 'approve') {
            return new PurchaseApprovedEvent();
        }

        return new PurchaseRejectedEvent();
    }
}
```

The lifecycle:

1. **Request** — the node builds an `InterruptRequest` describing the pause and calls `interrupt()`.
2. **Suspend** — the executor stops traversal, persists the workflow's steps, and marks the returned state as interrupted. The request travels outbound on the state for your application to render.
3. **Decision** — your application presents the request to a human and collects the answer.
4. **Resume** — you call `resume($payload)` on a workflow rebuilt with the same `runId`. Traversal replays: completed nodes are skipped, and the interrupted node re-runs with the payload injected — `interrupt()` returns it instead of suspending again.

#### The Persistence layer

Suspend & resume works by **replay**: every node executes as a durable step, and completed steps are persisted so a resumed run can skip straight to the interrupted node. That requires a persistence backend. Without one, there is nothing to resume from.

```php
use NeuronAI\Workflow\Persistence\FilePersistence;
use NeuronAI\Workflow\Persistence\PersistenceInterface;
use NeuronAI\Workflow\Workflow;

class MyWorkflow extends Workflow
{
    ...
    
    protected function persistence(): PersistenceInterface
    {
        return new FilePersistence(__DIR__);
        // or: new DatabasePersistence($pdo)
        // or: EloquentPersistence for Laravel apps
    }
}
```

The default `InMemoryPersistence` supports suspend & resume **within the same process** (useful in tests); use `FilePersistence`, `DatabasePersistence`, or `EloquentPersistence` when the resume happens in a later request or a background worker.

The request is fire-and-forget. The `InterruptRequest` itself is not persisted, only an "interrupted" flag is stored per step. On resume the node re-executes and rebuilds the request deterministically (replay-by-rerun).

Two consequences:

* You may safely put live object instances in a custom request, nothing is ever serialized.
* If your application needs to show "what is this run waiting for?" later, store the request yourself at suspend time (see below), the framework hands it to you through the returned state.

### Catching the interruption

After a suspend, your application has to detect that the workflow paused, show the request to a human, and keep a handle to come back later. Since `run()` returns normally, this is a plain check on the returned state: `isInterrupted()` tells you the workflow paused, `getInterruptRequest()` gives you the outbound request to render, and the `runId` is the resume token.

```php
use NeuronAI\Workflow\Persistence\FilePersistence;
use NeuronAI\Workflow\Persistence\PersistenceInterface;
use NeuronAI\Workflow\Workflow;

class MyWorkflow extends Workflow
{
    protected function nodes(): array
    {
        return [
            new FirstNode(),
            new ApprovalNode(),
            new FinalNode(),
        ];
    }
    
    protected function persistence(): PersistenceInterface
    {
        return new FilePersistence(__DIR__);
    }
}

// Run the workflow
$state = $workflow->setWorkflowId('wk_id')->run();

// Check if it was interrupted
if ($state->isInterrupted()) {
    $request = $state->getInterruptRequest();  // the OUTBOUND request — render it
    $runId = $workflow->getRunId();            // the resume token — store it

    // e.g. store it for your UI, along with the resume token
    $stmt = $pdo->prepare("INSERT INTO pending_approvals (run_id, request) VALUES (?, ?)");
    $stmt->execute([$runId, json_encode($request)]);
}
```

Every `InterruptRequest` is `JsonSerializable`, so `json_encode($request)` gives your frontend everything it needs to render the pause.

#### Resuming

Once you have the human's answer, rebuild the workflow with the **same `runId`** , and deliver the inbound payload. You never rebuild or pass back the request — the payload alone is the answer. `resume()` takes no step identifier: the framework finds the interrupted step by replaying.

```php
// A new process, a new HTTP request — hours later.
$workflow = Workflow::make(runId: $runId)->setWorkflowId('wk_id');

// The inbound payload — a plain array with the answer to the pause.
$finalState = $workflow->resume(['purchase_1' => 'approve']);
```

If the workflow suspends again downstream (multiple approval points), the returned state is interrupted again and the cycle repeats with the same `runId`.

### Conditional interruption

Use `interruptIf()` to suspend only when a condition holds. The condition can be a boolean or a callback; when it doesn't hold, the method returns `null` and execution continues.

```php
$payload = $this->interruptIf(
    $order->total > 1000,
    new ApprovalRequest(
        message: 'High-value order — approval required.',
        actions: [new Action('order_1', 'Approve order', "Total: {$order->total}$")],
    )
);

// Or evaluate lazily
$payload = $this->interruptIf(
    fn (): bool => $state->get('confidence', 1.0) < 0.5,
    new ApprovalRequest(/* ... */)
);
```

### Waiting for an external event

Sometimes the answer doesn't come from a human but from another system, a payment webhook, a document upload, a callback from a third-party API. For this case `awaitEvent()` suspends the workflow until an event with the given name is delivered. It's sugar over `interrupt()` with a built-in `WaitForEventRequest`, so no custom class is needed.

```php
class WaitForPaymentNode extends Node
{
    public function __invoke(
        OrderCreatedEvent $event, 
        WorkflowState $state,
        WorkflowResources $resources
    ): Event {
        // Suspend until 'payment.confirmed' is delivered.
        $payment = $this->awaitEvent('payment.confirmed');

        $state->set('transaction_id', $payment['transaction_id']);
        return new OrderPaidEvent();
    }
}
```

Delivering the event is just a resume from wherever the event lands in your application — typically a webhook controller:

```php
// Your webhook controller, when the payment provider calls back:
$workflow = OrderWorkflow::make(runId: $order->workflow_run_id)
    ->setPersistence($persistence);

$workflow->signal('payment.confirmed', ['transaction_id' => $webhook['tx_id']])->run();
```

The event name on the request is what your application uses to route the right event to the right run. The framework doesn't dispatch events itself; your code decides which suspended `runId` an incoming event belongs to.

### Custom interruption request

The built-in `ApprovalRequest` models approve/reject decisions on a list of actions. When your pause needs a different shape — say, a human editing generated content before it's saved — subclass an existing request type and add the outbound context your UI needs. Extend `WaitForEventRequest` (a human answer is an external event delivered to the workflow); you specialize the payload, not the pause category.

```php
namespace App\Neuron;

use NeuronAI\Workflow\Interrupt\WaitForEventRequest;

class ContentReviewRequest extends WaitForEventRequest
{
    public const EVENT_NAME = 'content.review';
    
    public function __construct(
        protected string $message,
        protected string $content,
        ?DateTimeImmutable $expiresAt = null,
    ) {
        parent::__construct(self::EVENT_NAME, $expiresAt);
    }

    public function getMessage(): string
    {
        return $this->message;
    }

    public function getContent(): string
    {
        return $this->content;
    }

    public function jsonSerialize(): array
    {
        return [
            'type' => $this->type(),
            'note' => $this->note,
            'content' => $this->content,
        ];
    }
}
```

The request only describes the pause and can be used to carry infomration your application need to present the interrupted process to the user.&#x20;

Use it in a node. The answer will come back as a payload:

```php
class ContentReviewNode extends Node
{
    public function __invoke(
        DraftEvent $event, 
        WorkflowState $state,
        WorkflowResources $resources
    ): Event {
        // Generate the article once, durably (see memoize below).
        $draft = $this->memoize('draft', fn (): string => ContentCreatorAgent::make()
            ->chat(new UserMessage($event->prompt))
            ->getMessage()
            ->getContent());

        // Suspend: send the draft OUT for review.
        $payload = $this->interrupt(
            new ContentReviewRequest('Review this article before publishing.', $draft)
        );

        // Resume: the edited text comes back IN as the payload.
        $state->set('content', $payload['edited_content']);

        return new PublishEvent();
    }
}
```

And on the application side:

```php
$state = $workflow->setWorkflowId('wk_id')->run();

if ($state->isInterrupted()) {
    /** @var ContentReviewRequest $request */
    $request = $state->getInterruptRequest();
    // render $request->getContent() in your editor UI...
}

// Later, deliver the edited text as the payload:
$finalState = $workflow->signal(
    event: ContentReviewRequest::EVENT_NAME, 
    payload: ['edited_content' => $editedText]
)->run();
```

### Consuming the feedback

The answer to the interruption is the **payload** you pass to the run() method, and the primary way to consume it is the return value of the suspend verb, exactly where the pause happened:

```php
$payload = $this->interrupt(new ContentReviewRequest(...));
// resuming: $payload is the delivered answer
```

If you need to branch _before_ reaching the interrupt call — for example to skip pre-interrupt logic entirely on resume — the node exposes the resume context directly:

```php
public function __invoke(
    InputEvent $event, 
    WorkflowState $state,
    WorkflowResources $resources
): Event {
    // isResuming() is true when this node run was triggered by resume():
    // the inbound payload has been injected and is readable up front.
    if ($this->isResuming()) {
        $payload = $this->getResumePayload();

        if (($payload['review_1'] ?? null) === 'approve') {
            $state->set('is_sufficient', true);
            return new OutputEvent();
        }
    }

    // First pass (or rejected): do the work and suspend.
    $this->interrupt(
        new ApprovalRequest(
            message: 'Should I continue?',
            actions: [new Action('review_1', 'Answer review', $state->get('review'))],
        )
    );

    return new InputEvent(); // unreachable on first pass; reached on rejected resume
}
```

In most nodes you won't need this: wrap the pre-interrupt work in `memoize()` and let the node re-run. The memoized work is recalled, not repeated, and the linear `interrupt()`-returns-the-answer style stays readable.

### Durable steps and `memoize()`

When a workflow resumes, the interrupted node **re-executes from the top** — any statement before the interrupt call would run again, re-billing an LLM call or re-sending an email. Two layers of durability protect you.

**Between nodes, you get durability for free.** Every node executes as a durable step: completed steps are persisted and skipped on replay. Nodes _before_ the interrupted one never re-run.

**Inside a node, use `memoize()`.** It executes a closure and persists its return value mid-node, before the node returns. When the node re-executes — on resume, or after a crash — the recorded value is returned _without_ running the closure again.

```php
class SentimentNode extends Node
{
    public function __invoke(
        ReviewEvent $event, 
        WorkflowState $state,
        WorkflowResources $resources
    ): Event {
        // Runs at most once, even across resume/crash replays of this node.
        $sentiment = $this->memoize('sentiment', fn (): SentimentResult => MyAgent::make()
            ->structured(new UserMessage($event->review), SentimentResult::class));

        if ($sentiment->isNegative()) {
            $payload = $this->interrupt(
                new ApprovalRequest(
                    message: 'Negative review detected. Should I answer it?',
                    actions: [new Action('review_1', 'Answer review', $sentiment->content)],
                )
            );

            // On resume the memoized $sentiment was recalled instantly above,
            // and interrupt() returned the answer here.
            if (($payload['review_1'] ?? null) === 'approve') {
                return new AnswerReviewEvent();
            }
        }

        return new SkipReviewEvent();
    }
}
```

`memoize()` takes:

* a **name**, unique within the node (the framework scopes it to the specific node execution automatically);
* a **Closure** wrapping the work whose result must survive a re-run.

> `checkpoint()` from the previous major version is deprecated and now delegates to `memoize()`. Unlike the old in-memory checkpoint, `memoize()` is **durable**: the value is persisted, so it also protects against crashes — not just interruptions.
