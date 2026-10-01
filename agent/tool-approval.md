---
description: Review, approve, or deny selected tool calls before they execute.
---

# Tool Approval

{% hint style="warning" %}
#### Coding Agent Skill

Use **`/neuron-tool-approval`** to teach your coding agent how to implement full tool approval flow in your application.

[AI-Assisted Development](../overview/agentic-development.md)
{% endhint %}

Neuron provides you with full support for the human in the loop patterns including tool approval. The framework intercepts the tool call and pause, waiting for the user's final decision. To activate tool approval you need to equip your Agent with two components:

* A durable [MessageStore](chat-history-and-memory.md#message-store)
* The [Persistence](../workflow/persistence.md) component

```php
use NeuronAI\Agent\Middleware\ToolApproval;
use NeuronAI\Chat\History\FileChatHistory;
use NeuronAI\Workflow\Persistence\FilePersistence;

class MyAgent extends Agent
{
    ...
    
    /**
     * Register tools
     */
    protected function tools(): array
    {
        return [
            BuyTicketTool::make(),
        ];
    }
    
    /**
     * Durable chat history
     */
    protected function chatHisotry(): ChatHistoryInterface
    {
        return new FileChatHistory(
            directory: '/path-to-filesystem',
            key: 'conversation ID',
        );
    }
    
    /**
     * Workflow persistence
     */
    protected function persistence(): PersistenceInterface
    {
        return new FilePersistence(
            directory: '/path-to-filesystem'
        );
    }
}
```

The framework will manage the entire approval flow automatically. You only have to take care of the User Interface. But before dive deeper into this process let's see how you can define approval policies for your tools.

Just override the `approvalPolicy()` method on the Tool class to determine whether the tool shuold be gated for approval. The method will receive the inputs the model want to use to call the tool. Returning a string counts as true and doubles as the approval request reason.

```php
class BuyTicketTool extends Tool
{
    ...,
    
    protected function approvalPolicy(): bool|string
    {
        // The tool requires approval if the amount is greather than 100
        return $this->getInput('amount') > 100;
    }
}
```

You can also override the default policy when you attach the tool to the agent:

```php
class MyAgent extends Agent
{
    ...,
    
    protected function tools(array $inputs): array
    {
        return [
            BuyTicketTool::make()->requireApproval(true|false),
            
            /*
             * by the time the callback runs, the tool instance it receives 
             * is not the bare registry tool, it's a clone with this call's inputs 
             * already bound.
             */
            BuyTicketTool::make()->withApprovalPolicy(
                fn(ToolInterface $tool): bool|string => $tool->getInput('amount') > 100
            )
        ];
    }
}
```

### Managing The Approval Flow

**Chat history is the system of record for tool approval.** Everything your UI needs: which tools are awaiting a decision, _why_ each one is asking, and what was already decided, lives on the **last message of the thread**.&#x20;

While an approval is pending, the framework guarantees the last `ToolCallMessage` is the tail of the thread. So your UI rendering logic only ever looks at this last message.

#### Detecting Approval on UI

When you load a thread, check the last message:

1. `type` is `"tool_call"`
2. at least one entry in `tools[]` has `"approval": "pending"`

If both hold, the conversation is suspended: render the approval UI and **lock the input box**, no new user messages can be sent until all decisions have been taken.

```javascript
const last = thread.messages.at(-1);
const pendingTools = last?.type === 'tool_call'
    ? last.tools.filter(t => t.approval === 'pending')
    : [];
const isSuspended = pendingTools.length > 0;
```

#### The JSON you'll deal with

This is a real serialized `ToolCallMessage` awaiting approval: two gated tools, one still pending, one already rejected in an earlier partial submission:

```json
{
    "__id": "msg_7488585683808067584",
    "resume_token": "workflow_6650a1b2c3d4e",
    "role": "assistant",
    "content": [],
    "type": "tool_call",
    "tools": [
        {
            "callId": "toolu_01A2B3C4D5E6F7",
            "name": "delete_file",
            "description": "Delete a file from the filesystem",
            "parameters": [],
            "inputs": { "path": "C:/old_logs.txt" },
            "result": null,
            "approval": "pending",
            "approvalReason": "Deleting a file is irreversible",
            "rejectReason": null
        },
        {
            "callId": "toolu_08G9H0I1J2K3L4",
            "name": "send_email",
            "description": "Send an email to a recipient",
            "parameters": [],
            "inputs": { "to": "team@example.com", "subject": "Logs cleanup" },
            "result": null,
            "approval": "rejected",
            "approvalReason": "Outbound email reaches people outside this workspace",
            "rejectReason": "Do not email the whole team"
        }
    ]
}
```

&#x20;`approvalReason` is the _tool talking to the human_ ("here's why I'm asking"), `rejectReason` is the _human talking back to the model_ ("here's why I said no").

#### Rendering guidance

* Render one card per entry in `tools[]` that **has** an `approval` field: tool name, its `inputs`, the `approvalReason` if present, and Approve / Deny actions. Deny should offer an optional free-text reason — the model receives it verbatim, so a good reason produces a better model response.
* Tools already `"approved"`/`"rejected"` are shown as decided — but remain **revisable until the full set is decided** (last-write-wins). Once the last decision lands, the workflow proceeds immediately and the states become historical.
* Approvals are bare — there's no "approve with comment".

### One endpoint for the whole conversation

A normal turn and an approval resume are the same operation from the endpoint's point of view: build the agent from the thread, feed it what the client sent, and return the thread's new state. The framework sorts out the rest — with a `payload`, the resume token is adopted from the chat history tail; with a message, a fresh turn starts.

Your endpoint needs only the `threadId`:

```php
class ChatController
{
    public function chat(ChatRequest $request, string $threadId)
    {
        $agent = MyAgent::make($threadId);
    
        $state = $request->has('decisions')
            ? $agent->submitApprovalDecisions($request->get('decisions'))->run()
            : $agent->chat(new UserMessage($request->get('message')));
        
        return [
            'status'  => $state->isInterrupted() ? 'awaiting_approval' : 'completed',
            'message' => $state->getMessage()?->jsonSerialize(),
        ];
    }
}
```

### Submitting decisions

Thread suspended with a pending tools. Here is the body you need to send to resolve the waiting approval:

```
POST /threads/thread_42/chat
    { "decisions": { "toolu_01A2B3C4D5E6F7": "approve" } }
```

Decisions travel as a plain map keyed by the tool's `callId`. Each value is one of exactly three forms:

| You Send                   | Meaning                      | What the model sees                                                                            |
| -------------------------- | ---------------------------- | ---------------------------------------------------------------------------------------------- |
| "approve"                  | Run the tool                 | The tool's real output, nothing else. Approvals are bare — there is no "approve with comment". |
| "reject"                   | Skip the tool                | The rejection template with _"No specific instruction provided."_                              |
| \["reject", "your reason"] | Skip the tool, with feedback | The rejection template with your reason verbatim.                                              |
