---
description: Control and customize agent execution at every step
---

# Middleware

### What is a Middleware

Middleware provides a way to hook the workflow execution and therefore also Agent and RAG, since they too are workflows.&#x20;

The core Workflow execution involves calling nodes based on the events returned by other nodes. Middleware exposes hooks to step inside `before` and `after`  the execution of nodes:

<figure><img src="../.gitbook/assets/middleware.png" alt=""><figcaption></figcaption></figure>

### What can middleware do?

<table data-view="cards"><thead><tr><th></th><th></th></tr></thead><tbody><tr><td><strong>Control</strong></td><td>Transform prompts, tool selection, and output formatting.</td></tr><tr><td><strong>Guardrails</strong></td><td>Add retries, rate limits, implement guardrails, prompt rejection.</td></tr><tr><td><strong>Monitor</strong></td><td>Track agent behavior with logging, analytics, and debugging.</td></tr></tbody></table>

### Creating Middleware

You can use the command below to create a middleware  class:

{% tabs %}
{% tab title="Unix" %}
```bash
./vendor/bin/neuron make:middleware CustomMiddleware
```
{% endtab %}

{% tab title="Windows" %}
```powershell
.\vendor\bin\neuron make:middleware CustomMiddleware
```
{% endtab %}
{% endtabs %}

A new class will be created in your project with the default middleware structure:

```php
<?php

namespace App\Neuron\Middleware;

class CustomMiddleware implements WorkflowMiddleware
{
    /**
     * Execute before the node runs.
     */
    public function before(NodeInterface $node, Event $event, WorkflowState $state): void
    {
        // ...
    }
    
    /**
     * Execute after the node runs.
     */
    public function after(NodeInterface $node, Event $result, WorkflowState $state): void
    {
        // ...
    }
}
```

### Registering Middleware

If you would like to assign middleware to specific nodes, you may override the  `middleware` method when defining the workflow:

```php
class MyWorkflow extends Workflow
{
    /**
     * Define the nodes middleware.
     */
    protected function middleware(): array
    {
        return [
            NodeOne::class => new CustomMiddleware(),
        ];
    }
}
```

You can also assign multiple middleware to a node, defining an array:

```php
class MyWorkflow extends Workflow
{
    /**
     * Define the nodes middleware.
     */
    protected function middleware(): array
    {
        return [
            NodeOne::class => [
                new CustomMiddleware(),
                new AnotherMiddleware(),
            ]
        ];
    }
}
```

### Global Middleware

If you want middleware to run before and after every node, you may append it to the global middleware stack in the workflow:

```php
class MyWorkflow extends Workflow
{
    /**
     * Define the global middleware.
     */
    protected function globalMiddleware(): array
    {
        return [
            new CustomMiddleware(),
        ];
    }
}
```

### Examples

Neuron provides prebuilt middleware for common use cases, like tool approval, or context summarization. You can check them out in the Agent section:

{% content-ref url="../agent/middleware.md" %}
[middleware.md](../agent/middleware.md)
{% endcontent-ref %}
