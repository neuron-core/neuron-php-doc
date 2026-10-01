---
description: Persist the Workflow State across executions.
metaLinks:
  alternates:
    - https://app.gitbook.com/s/GHx4l2LknIex7vFIUg1R/workflow/persistence
---

# Persistence

When we talk about persistence in Neuron, we're talking about the system's ability to capture and preserve the complete state of a running workflow at any moment.

Think of it like a sophisticated "save game" feature, but for business processes. At any point, when an interruption is asked from a node, Neuron create a snapshot of your workflow's state and store it in the persistence layer. Later, whether that's seconds, hours, or weeks, the workflow can be restored to exactly where it left of and continue as if nothing happened.

As usual in Neuron the Workflow persistence layer is built on top of a common interface so it's extensible and interchangeable. Below the supported persistence layer.

### InMemoryPersistence

It keep data in memory only for the current execution cycle.

```php
use NeuronAI\Workflow\Persistence\InMemoryPersistence;
use NeuronAI\Workflow\Persistence\PersistenceInterface;

class MyWorkflow extends Workflow
{
    ...
    
    protected function persistence(): PersistenceInterface
    {
        return new InMemoryPersistence();
    }
}
```

### FilePersistence

It will store the Workflow data and state into a local file.

```php
use NeuronAI\Workflow\Persistence\FilePersistence;
use NeuronAI\Workflow\Persistence\PersistenceInterface;

class MyWorkflow extends Workflow
{
    ...
    
    protected function persistence(): PersistenceInterface
    {
        return new FilePersistence(__DIR__);
    }
}
```

### RedisPersistence (_recommended_)

Use Redis, or Redis compatible, datastore to persist transient workflow execution data.

{% hint style="warning" %}
This component requires you to have the PHP Redis extension [`phpredis`](https://github.com/phpredis/phpredis) installed in your system.
{% endhint %}

```php
use NeuronAI\Workflow\Persistence\RedisPersistence;
use NeuronAI\Workflow\Persistence\PersistenceInterface;

class MyWorkflow extends Workflow
{
    ...
    
    protected function persistence(): PersistenceInterface
    {
        return new RedisPersistence(
            client: new Redis(...)
        );
    }
}
```

### DatabasePersistence

To persist the workflow interruption in the database you need to pass a `PDO` instance. If you are working on top of a framework you can easily get it from the ORM in the same way of the [SQLChatHistory](../agent/chat-history-and-memory.md#sqlchathistory) for chat history persistence.

```php
use NeuronAI\Workflow\Persistence\DatabasePersistence;
use NeuronAI\Workflow\Persistence\PersistenceInterface;

class MyWorkflow extends Workflow
{
    ...
    
    protected function persistence(): PersistenceInterface
    {
        return new DatabasePersistence(
            pdo: new \PDO(...)
        );
    }
}
```

Here are the SQL scripts to create the table:

{% tabs %}
{% tab title="MySQL/MariaDB" %}
```sql
CREATE TABLE workflow_store (
    `partition` VARCHAR(255) NOT NULL,
    `key`       VARCHAR(255) NOT NULL,
    `value`     TEXT NOT NULL,
    created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`partition`, `key`)
);
```
{% endtab %}

{% tab title="PostgreSQL/SQLite" %}
```sql
CREATE TABLE workflow_store (
    "partition" VARCHAR(255) NOT NULL,
    "key"       VARCHAR(255) NOT NULL,
    "value"     TEXT NOT NULL,
    created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY ("partition", "key")
);
```
{% endtab %}
{% endtabs %}

### EloquentPersistence

You should create your own Eloquent model and pass the class string as the constructor argument. The model can have custom relations, scopes, attributes, etc. but the basic structure must be based on this migration script:

```bash
php artisan make:migration create_workflow_interrupts_table --create=workflow_interrupts
```

```php
Schema::create('workflow_store', function (Blueprint $table) {
    $table->id();
    $table->string('partition');
    $table->string('key');
    $table->longText('value');
    $table->timestamps();
    
    $table->unique(['partition', 'key']);
});
```

#### WorkflowInterrupt model

This is the minimal required structure:

```php
class WorkflowStep extends Model
{
    protected $fillable = ['partition', 'key', 'value'];
}
```

Use it in the Workflow:

```php
use App\Models\WorkflowInterrupt;
use NeuronAI\Workflow\Persistence\EloquentPersistence;
use NeuronAI\Workflow\Persistence\PersistenceInterface;

class MyWorkflow extends Workflow
{
    ...
    
    protected function persistence(): PersistenceInterface
    {
        return new EloquentPersistence(WorkflowInterrupt::class);
    }
}
```
