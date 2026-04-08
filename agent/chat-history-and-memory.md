---
description: Learn how Neuron AI manage multi turn conversations.
---

# Chat History

Neuron AI provides you with a built-in system to manage the memory of a chat session you perform with the agent.

In many Q\&A applications you can have a back-and-forth conversation with the LLM, meaning the application needs some sort of "memory" of past questions and answers, and some logic for incorporating those into its current thinking.

For example, if you ask a follow-up question like "Can you elaborate on the second point?", this cannot be understood without the context of the previous messages.

In the example below you can see how the Agent doesn't know my name initially:

```php
use NeuronAI\Agent\Agent;
use NeuronAI\Chat\Messages\UserMessage;

$message = Agent::make()
    ->chat(new UserMessage("What's my name?"))
    ->getMessage();

echo $message->getContent();
// I'm sorry I don't know your name. Do you want to tell me more about yourself?
```

Clearly the Agent doesn't have any context about me. Now I try present me in the first message, and then ask for my name:

```php
use NeuronAI\Agent\Agent;
use NeuronAI\Chat\Messages\UserMessage;

$agent = Agent::make()

$message = $agent->chat(new UserMessage("Hi, my name is Valerio!"))->getMessage();
echo $message->getContent();
// Hi Valerio, nice to meet you, how can I help you today?

$message = $agent->chat(new UserMessage("Do you remember my name?"))->getMessage();
echo $message->getContent();
// Sure, your name is Valerio!
```

## How Chat History works

Neuron Agent takes the list of messages exchanged between your application and the LLM into an object called Chat History. It's a crucial part of the framework because the chat history needs to be managed based on the context window of the underlying LLM.

It's important to send past messages back to LLM to keep the context of the conversation, but if the list of messages grows enough to exceed the context window of the model the request will be rejected by the AI provider, because it exceeds the maximum capability of the LLM.

Chat history automatically truncates the list of messages to never exceed the context window avoiding unexpected errors. You may want to consider implementing more sophisticated context management strategies, like [summarization](middleware.md#summarization).

While cutting, the chat history tries to minimize the context loss. The internal trimmer can identify a cutting point slightly less aggressive than the initially identified. So, to make sure the agent conversation stays in the limit, **you should configure the context window in the agent chat history with a margin of 5%-10% from the actual limit of the underlying model**.

If your model works with a 200K context window, you should instantiate your chat history with 190K for example.

```php
namespace App\Neuron;

use NeuronAI\Agent\Agent;
use NeuronAI\Providers\AIProviderInterface;
use NeuronAI\Chat\History\ChatHistoryInterface;
use NeuronAI\Chat\History\InMemoryChatHistory;

class MyAgent extends Agent
{
    protected function provider(): AIProviderInterface
    {
        ...
    }
    
    protected function chatHistory(): ChatHistoryInterface
    {
        return new InMemoryChatHistory(
            contextWindow: 190000
        );
    }
}
```

## How to feed a previous conversation

Sometimes you already have a representation of user to assistant conversation and you need a way to feed the agent with previous messages.

You can just pass an array of messages to the `chat()` method. This conversation will be automatically loaded into the agent memory and you can continue to iterate on it.

```php
use NeuronAI\Chat\Enums\MessageRole;
use NeuronAI\Chat\Messages\Message;

$message = MyAgent::make()
    ->chat([
        new Message(MessageRole::USER, "Hi, my company is called Inspector.dev"),
        new Message(MessageRole::ASSISTANT, "Great, how can I assist you today?"),
        new Message(MessageRole::USER, "What's the name of the company I work for?"),
    ])
    ->getMessage();
    
echo $message->getContent();
// You work for Inspector.dev
```

The last message in the list will be considered the most recent.

## Register the chat history

By default Neuron Agent uses an "in memory" chat history. That means it keeps messages only for the current execution cycle. But, if you want to persist messages across sessions you can tell the agent to use a different component by implementing the `chatHistory` method in the Agent class.

```php
namespace App\Neuron;

use NeuronAI\Agent\Agent;
use NeuronAI\Providers\AIProviderInterface;
use NeuronAI\Chat\History\ChatHistoryInterface;
use NeuronAI\Chat\History\InMemoryChatHistory;

class MyAgent extends Agent
{
    protected function provider(): AIProviderInterface
    {
        ...
    }
    
    protected function chatHistory(): ChatHistoryInterface
    {
        return new InMemoryChatHistory(
            contextWindow: 50000
        );
    }
}
```

## Available Chat History Implementations

### InMemoryChatHistory

It simply store the list of messages into an array. It is kept in memory only during the current execution. It's used by default if you don't explicitly register another component.

```php
namespace App\Neuron;

use NeuronAI\Agent\Agent;
use NeuronAI\Chat\History\ChatHistoryInterface;
use NeuronAI\Chat\History\InMemoryChatHistory;
use NeuronAI\Providers\AIProviderInterface;

class MyAgent extends Agent
{
    ...
    
    protected function chatHistory(): ChatHistoryInterface
    {
        return new InMemoryChatHistory(
            contextWindow: 150000
        );
    }
}
```

### FileChatHistory

This compnent makes you able  to persist the ongoing conversation with the agent in a file, and resume it later in time. To create an instance of the `FileChatHistory` you need to pass the absolute path of the `directory` where you want to store conversations, and the unique `key` for the current conversation.

```php
namespace App\Neuron;

use NeuronAI\Agent\Agent;
use NeuronAI\Chat\History\ChatHistoryInterface;
use NeuronAI\Chat\History\FileChatHistory;
use NeuronAI\Providers\AIProviderInterface;

class MyAgent extends Agent
{
    ...
    
    protected function chatHistory(): ChatHistoryInterface
    {
        return new FileChatHistory(
            directory: '/home/app/storage/neuron',
            key: 'THREAD_ID',
            contextWindow: 150000
        );
    }
}
```

The `key` parameter allows you to store different files to separate conversations. You can use a unique key for each user, or the ID of a thread to make users able to store multiple conversations.

### SQLChatHistory

This component allows you to store the ongoing conversation into a SQL database. Before using this component you must create the table on your database to store messages. Here is the SQL script:

```sql
CREATE TABLE IF NOT EXISTS chat_history (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  thread_id VARCHAR(255) NOT NULL,
  messages LONGTEXT NOT NULL,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
 
  UNIQUE KEY uk_thread_id (thread_id),
  INDEX idx_thread_id (thread_id)
);
```

You can customize this table addind more columns eventually to add a relation to your users or similar use cases. You can also customize the table name passing your custom one when creating the instance.

To create an instance of the `SQLChatHistory` you need to pass the `thread_id` to separate different conversation threads, and the `PDO` connection to the database.

```php
namespace App\Neuron;

use NeuronAI\Agent\Agent;
use NeuronAI\Chat\History\ChatHistoryInterface;
use NeuronAI\Chat\History\SQLChatHistory;
use NeuronAI\Providers\AIProviderInterface;

class MyAgent extends Agent
{
    ...
    
    protected function chatHistory(): ChatHistoryInterface
    {
        return new SQLChatHistory(
            thread_id: 'THREAD_ID',
            pdo: new \PDO("mysql:host=localhost;dbname=DB_NAME;charset=utf8mb4", "DB_USER", "DB_PASS"),
            table: 'chat_hisotry',
            contextWindow: 150000
        );
    }
}
```

If your application is built on top of a framewrok you can easily get the PDO connection from the ORM. Here are is couple of examples in the context of Laravel or Symfony applications.

#### Laravel

```php
namespace App\Neuron;

use NeuronAI\Agent\Agent;
use NeuronAI\Chat\History\ChatHistoryInterface;
use NeuronAI\Chat\History\SQLChatHistory;
use NeuronAI\Providers\AIProviderInterface;

class MyAgent extends Agent
{
    ...
    
    protected function chatHistory(): ChatHistoryInterface
    {
        return new SQLChatHistory(
            thread_id: 'CHAT_THREAD_ID',
            pdo: \DB::connection()->getPdo(),
            table: 'chat_hisotry',
            contextWindow: 150000
        );
    }
}
```

#### Symfony

You can register your agent as a service with an instance of `Doctrine\DBAL\Connection` as a constructor dependency:

```php
namespace App\Neuron;

use Doctrine\DBAL\Connection;
use NeuronAI\Agent\Agent;
use NeuronAI\Chat\History\ChatHistoryInterface;
use NeuronAI\Chat\History\SQLChatHistory;
use NeuronAI\Providers\AIProviderInterface;

class MyAgent extends Agent
{
    public function __construct(protected Connection $connection)
    {}
    
    protected function chatHistory(): ChatHistoryInterface
    {
        return new SQLChatHistory(
            thread_id: 'CHAT_THREAD_ID',
            pdo: $this->connection->getNativeConnection(),
            table: 'chat_hisotry',
            contextWindow: 150000
        );
    }
}
```

### EloquentChatHisotry

You should create your own Eloquent model and pass the class string as the constructor argument. The model can have custom relations, scopes, attributes, etc. but the basic structure must be based on this migration script:

```bash
php artisan make:migration create_chat_messages_table --create=chat_messages
```

```php
Schema::create('chat_messages', function (Blueprint $table) {
     $table->id();
     $table->string('thread_id')->index();
     $table->string('role');
     $table->string('content');
     $table->string('meta')->nullable();
     $table->timestamps();

     $table->index(['thread_id', 'id']); // For efficient ordering and trimming
});
```

#### ChatMessage model example

```php
class ChatMessage extends Model
{
    protected $fillable = [
        'thread_id', 'role', 'content', 'meta'
    ];
    
    protected $casts = [
        'content' => 'array', 
        'meta' => 'array'
    ];
    
    /**
     * return BelongsTo<Conversation, $this>
     */
    public function conversation(): BelongsTo
    {
        return $this->belongsTo(Conversation::class, 'thread_id');
    }
}
```

Use in your agent:

```php
namespace App\Neuron;

use NeuronAI\Agent\Agent;
use NeuronAI\Chat\History\ChatHistoryInterface;
use NeuronAI\Chat\History\EloquentChatHistory;

class MyAgent extends Agent
{
    ...
    
    protected function chatHistory(): ChatHistoryInterface
    {
        return new EloquentChatHistory(
            thread_id: 'THREAD_ID',
            modelClass: ChatMessage::class,
            contextWindow: 150000
        );
    }
}
```

## Implement custom chat history

You can create a custom implementation of the chat history to support different persistent layer just implementing `AbstractChatHistory`. It allows you to inherit several behaviors for the internal history management, so you have just to implement a couple of methods to save messages into the storage system you want to use.

```php
abstract class AbstractChatHistory implements ChatHistoryInterface
{
    /**
     * @param Message[] $messages
     */
    protected function setMessages(array $messages): void
    {
        // Handle saving the entire history at once every time the history is updated.
    }

    protected function onNewMessage(Message $message): void
    {
        // Handle single message addition
    }

    protected function onTrimHistory(int $index): void
    {
        // When the trim is triggered, 
        // the messages in the position from zero to $index must be removed.
    }

    protected function clear(): void
    {
        // Remove all messages.
    }
}
```

The abstract class already implement some utility methods to calculate tokens usage based on the AI provider responses and automatically cut the conversation based on the size of the context window. You just have to focus on the interaction with the underlying storage to add and remove messages, or clear the entire history.

We strongly suggest to look at other implementations like `FileChatHistory` to understand how to create your own.

### Serialize/Deserialize Messages

When the ChatHistory needs to store a message it must be serialized. The same way, when the ChatHistory component is instantiated it should load all the previous messages from the underlying storage (database, cache, etc) and deserialize them to the original message type.&#x20;

To serialize/deserialize messages consistently the `AbstractChatHistory` provides you with `serializeMessage()` and `deserializeMessage()` methods. Here is an example of how to use them in an hypothetical database chat history implementation:

```php
<?php

namespace NeuronAI\Chat\History;

use NeuronAI\Chat\Messages\Message;

class DatabaseChatHistory extends AbstractChatHistory
{
    public function __construct(protected \PDO $db) 
    {
        // Retrieve the current conversation from the underlying storage
        $messages = $this->db->select(...);
        
        // Deserialize properly initialize the correct message types with the correct data.
        $this->history = $this->deserializeMessages($messages);
    }

    protected function onNewMessage(Message $message): void
    {
        // Store the serialized version.
        $this->db->insert($message->jsonSerialize());
    }

    ...
}
```
