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

Persistence comes into play when you intend to use interruption (e.g. [Tool Approval](../agent/middleware.md#tool-approval-human-in-the-loop)).

### InMemoryPersistence

It keep data in memory only for the current execution cycle.

```php
use NeuronAI\Workflow\Persistence\InMemoryPersistence;

$workflow = new WorkflowAgent(
    new InMemoryPersistence()
);
```

### FilePersistence

It will store the Workflow data and state into a local file.

```php
use NeuronAI\Workflow\Persistence\FilePersistence;

$workflow = new WorkflowAgent(
    new FilePersistence(__DIR__), 
);
```

### Database

To persist the workflow interruption in the database you need to pass a `PDO` instance. If you are working on top of a framework you can easily get it from the ORM in the same way of the [SQLChatHistory](../agent/chat-history-and-memory.md#sqlchathistory).

```php
use NeuronAI\Workflow\Persistence\DatabasePersistence;

$workflow = new WorkflowAgent(
    new DatabasePersistence(
        pdo: new \PDO(...),
        table: 'workflow_interrupts'
    ), 
);
```

Here are the SQL scripts to create the table:

{% tabs %}
{% tab title="MySQL/MariaDB" %}
```sql
CREATE TABLE IF NOT EXISTS workflow_interrupts (
    workflow_id VARCHAR(255) PRIMARY KEY,
    interrupt LONGBLOB NOT NULL,
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
    interrupt BYTEA NOT NULL,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL
);

CREATE INDEX idx_workflow_id ON workflow_interrupts(workflow_id);
CREATE INDEX idx_updated_at ON workflow_interrupts(updated_at);
```
{% endtab %}

{% tab title="SQLite" %}
```sql
CREATE TABLE workflow_interrupts (
    workflow_id TEXT PRIMARY KEY,
    interrupt BLOB NOT NULL,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL
);

CREATE INDEX idx_workflow_id ON workflow_interrupts(workflow_id);
CREATE INDEX idx_updated_at ON workflow_interrupts(updated_at);
```
{% endtab %}
{% endtabs %}

### Eloquent

You should create your own Eloquent model and pass the class string as the constructor argument. The model can have custom relations, scopes, attributes, etc. but the basic structure must be based on this migration script:

```bash
php artisan make:migration create_workflow_interrupts_table --create=workflow_interrupts
```

```php
Schema::create('workflow_interrupts', function (Blueprint $table) {
    $table->id();
    $table->string('workflow_id')->unique();
    $table->longText('interrupt')->charset('binary');
    $table->timestamps();
});
```

#### WorkflowInterrupt model

This is the minimal required structure:

```php
class WorkflowInterrupt extends Model
{    
    protected $fillable = ['workflow_id', 'interrupt'];
}
```

Use with Workflow:

```php
use App\Models\WorkflowInterrupt;
use NeuronAI\Workflow\Persistence\EloquentPersistence;

// Creating a workflow
$workflow = WorkflowAgent(
    persistence: new EloquentPersistence(WorkflowInterrupt::class)
);
```
