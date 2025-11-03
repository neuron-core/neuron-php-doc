---
description: Persist the Workflow State across executions.
---

# Persistence

When we talk about persistence in Neuron, we're talking about the system's ability to capture and preserve the complete state of a running workflow at any moment. This includes:

* **All variables and their current values**&#x20;
* **The exact execution position** – which node is active, which have completed, which are waiting
* **Context and metadata** – timestamps, user information, decision history
* **Error states and retry counters** – so failures can be handled gracefully

Think of it like a sophisticated "save game" feature, but for business processes. At any point, when an interruption is asked from a node, Neuron create a snapshot of your workflow's state and store it in the persistence layer. Later – whether that's seconds, hours, or weeks – the workflow can be restored to exactly that moment and continue as if nothing happened.

As usual in Neuron the Workflow persistence layer is built on top of a common interface so it's extensible and interchangeable. Below the supported persistence layer.

### When to use Persistence

Persistence comes into play when you intend to use interruption. The persistence component requires to decalre also a workflow ID.

### InMemoryPersistence

It keep data in memory only for the current execution cycle.

```php
$workflow = new WorkflowAgent(
    new InMemoryPersistence(), 
    'CUSTOM_ID'
);
```

### FilePersistence

It will store the Workflow data and state into a local file.

```php
$workflow = new WorkflowAgent(
    new FilePersistence(__DIR__), 
    'CUSTOM_ID'
);
```

{% hint style="warning" %}
_FilePersistence_ component uses PHP serialization to store the current state of the Workflow. While this allows you to use any PHP object as an item of the Workflow state (e.g. [ChatHistory](../the-basics/chat-history-and-memory.md)), it also has some limitations like it does not support serialization of Closure. If objects you want to save in the Workflow state conflict with the PHP standard serialization process, you can implement the [Serializable interface](https://www.php.net/manual/en/class.serializable.php) to let the NeuronAI persistence component know of how to serialize the object in the correct way.
{% endhint %}

### DatabasePersistence

To persist the workflow interruption in the database you need to pass a `PDO` instance. If you are working on top of a framework you can get it from the ORM in the same way of the [SQLChatHistory](../the-basics/chat-history-and-memory.md#sqlchathistory).

```php
use NeuronAI\Workflow\Persistence\DatabasePersistence;

$workflow = new WorkflowAgent(
    new DatabasePersistence(
        pdo: new \PDO(...),
        table: 'workflow_interrupts'
    ), 
    'CUSTOM_ID'
);
```

Here are the SQL scripts to create the table:

{% tabs %}
{% tab title="MySQL/MariaDB" %}
```sql
CREATE TABLE IF NOT EXISTS workflow_interrupts (
    workflow_id VARCHAR(255) PRIMARY KEY,
    data LONGBLOB NOT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    
    INDEX idx_workflow_id (workflow_id),
    INDEX idx_updated_at (updated_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```
{% endtab %}

{% tab title="PostgreSQL" %}
```sql
CREATE TABLE workflow_interrupts (
    workflow_id VARCHAR(255) PRIMARY KEY,
    data BYTEA NOT NULL,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL
);

CREATE INDEX idx_workflow_id ON workflow_interrupts(workflow_id);
CREATE INDEX idx_updated_at ON workflow_interrupts(updated_at);
```
{% endtab %}

{% tab title="SQLite" %}
```ruby
message = "hello world"
puts message
```
{% endtab %}
{% endtabs %}
