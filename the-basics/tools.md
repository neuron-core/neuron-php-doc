---
description: >-
  Give Agents the ability to interact with your application context and
  services.
---

# Tools & Toolkits

The core agent loop involves calling a model, letting it choose tools to execute, and then finishing when no more tools are needed to provide a response:

<figure><img src="../.gitbook/assets/NeuronAI.png" alt=""><figcaption></figcaption></figure>

### What is a Tool

Tools enable Agents to go beyond generating text by facilitating interaction with your application services, or external APIs.

Think about Tools as special functions that your AI agent can use when it needs to perform specific tasks. They let you extend your Agent's capabilities by giving it access to specific functions it can call inside your code.

{% embed url="https://www.youtube.com/watch?v=lI8xE-uIek8" %}

In the [YouTubeAgent](agent.md) example we can define a tool to make the Agent able to retrieve the YouTube video transcription, so it can crteate a short summary:

```php
namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\SystemPrompt;
use NeuronAI\Providers\AIProviderInterface;
use NeuronAI\Providers\Anthropic\Anthropic;
use NeuronAI\Tools\PropertyType;
use NeuronAI\Tools\Tool;
use NeuronAI\Tools\ToolProperty;

class YouTubeAgent extends Agent
{
    protected function provider(): AIProviderInterface
    {
        // return an AI provider instance (Anthropic, OpenAI, Ollama, Gemini, etc.)
        return new Anthropic(
            key: 'ANTHROPIC_API_KEY',
            model: 'ANTHROPIC_MODEL',
        );
    }
    
    public function instructions(): string 
    {
        return (string) new SystemPrompt(
            background: ["You are an AI Agent specialized in writing YouTube video summaries."],
            steps: [
                "Get the url of a YouTube video, or ask the user to provide one.",
                "Use the tools you have available to retrieve the transcription of the video.",
                "Write the summary.",
            ],
            output: [
                "Write a summary in a paragraph without using lists. Use just fluent text.",
                "After the summary add a list of three sentences as the three most important take away from the video.",
            ]
        );
    }
    
    protected function tools(): array
    {
        return [
            Tool::make(
                'get_transcription',
                'Retrieve the transcription of a youtube video.',
            )->addProperty(
                new ToolProperty(
                    name: 'video_url',
                    type: PropertyType::STRING
                    description: 'The URL of the YouTube video.',
                    required: true
                )
            )->setCallable(function (string $video_url) {
                return "Video transcripton...";
            })
        ];
    }
}

```

Let’s break down the code.

We introduced the new method `tools()` into the Agent class. This method expects to return an array of Tool objects that the AI will be able to use if needed.

In this example we return an array of just one tool, named `get_transcription`.&#x20;

Notice that the `ToolProperty` we define should match with the signature of the function you use as a callable. The callable gets the `$video_url` arguments, and the name of the property is exactly "video\_url".

The most important thing are the name and description you give to the tool and its properties. All these pieces of information will be passed to the LLM in natural language. The more explicit and clear you are, the more likely the LLM understands when, if, and why, it’s the case to use the tool.

Once the Agent decides to use a tool the callable function is executed. Here we can implement the logic to retrieve the video transcription and return the information back to the LLM.

Neuron provides you with these clear and simple APIs and automates all the underlying interactions with the LLM. Once you get the point it can immediately open to a possibility to connect basically everything you want to the Agent. Being able to execute local functions allows you to invoke any external APIs or application components.

### Custom Tools

Thanks to the Neuron modular architecture, Tools are components that implement  `ToolInterface` . You are free to create pre-packaged tool classes to make the agent able to perform sapecific actions, and release them as external composer packages or submit a PR to our repository to have them integrated into the core framework.

To create a new Tool execute the console command below:

{% tabs %}
{% tab title="Unix" %}
```bash
php vendor/bin/neuron make:tool App\\Neuron\\GetTranscriptionTool
```
{% endtab %}

{% tab title="Windows" %}
```powershell
php .\vendor\bin\neuron make:tool App\Neuron\GetTranscriptionTool
```
{% endtab %}
{% endtabs %}

You can customize the scaffolding of the tool with the code below:

```php
<?php

namespace App\Neuron\Tools;

use GuzzleHttp\Client;
use NeuronAI\Tools\PropertyType;
use NeuronAI\Tools\Tool;
use NeuronAI\Tools\ToolProperty;

class GetTranscriptionTool extends Tool
{
    protected Client $client;
    
    public function __construct(protected string $key)
    {
        // Define Tool name and description
        parent::__construct(
            'get_transcription',
            'Retrieve the transcription of a youtube video.',
        );
    }
    
    /**
     * Return the list of properties.
     */
    protected function properties(): array
    {
        return [
            new ToolProperty(
                name: 'video_url',
                type: PropertyType::STRING,
                description: 'The URL of the YouTube video.',
                required: true
            )
        ];
    }
    
    /**
     * Implementing the tool logic
     */
    public function __invoke(string $video_url): string
    {
        $response = $this->getClient()
            ->get('transcript?url=' . $video_url.'&text=true')
            ->getBody()
            ->getContents();

        $response = json_decode($response, true);

        return $response['content'];
    }
    
    protected function getClient(): Client
    {
        return $this->client ?? $this->client = new Client([
            'base_uri' => 'https://api.supadata.ai/v1/youtube/',
            'headers' => [
                'x-api-key' => $this->key,
            ]
        ]);
    }
}
```

**Tool name and description**: Define name and description of the tool in the tool constructor. Invest in prompt engineering to help the model take better decisions.

**The properties method**: Implement this method to return the list of properties the tool expects.

**The `__invoke` method**: Here you need to implement the logic of the tool, and return a result that will be returned back to the model. The PHP `__invoke` magic method is used by default.

Notice how the `__invoke()` method accepts the same arguments defined by the `ToolProperty` . In this example I'm using an external service to retrieve the YouTube video transcription called [Supadata.ai](https://supadata.ai/).

You can attach the tool in the agent class as usual:

```php
<?php

namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\Providers\AIProviderInterface;
use NeuronAI\Providers\Anthropic\Anthropic;
use NeuronAI\SystemPrompt;
use App\Neuron\Tools\MyCustomTool;

class YouTubeAgent extends Agent
{
    public function provider(): AIProviderInterface
    {
        // return an AI provider instance (Gemini, OpenAI, Ollama, etc.)
        return new Anthropic(
            key: 'ANTHROPIC_API_KEY',
            model: 'ANTHROPIC_MODEL',
        );
    }
    
    public function instructions(): string
    {
        return (string) new SystemPrompt(...);
    }
    
    public function tools(): array
    {
        return [
            GetTranscriptionTool::make('API_KEY'),
        ];
    }
}
```

GetTranscriptions is just an example. You can eventually implement other tools to make the Agent able to retrieve other video metadata to enhance its video analysis capabilities.

Finally you can talk to the agent asking for the summary of a YouTube video.

```php
use NeuronAI\Chat\Messages\UserMessage;

$response = YouTubeAgent::make($user)->chat(
    new UserMessage('What about this video: https://www.youtube.com/watch?v=WmVLcj-XKnM')
);
    
echo $response->getContent();

/**

Based on the transcription, I'll provide a summary of this powerful environmental 
message from "Mother Nature":
This video presents ...

Three most important takeaways:

1. Nature has existed ...

2. The wellbeing of humanity is ...

3. How humans choose to act toward Nature determines ...

*/
```

### Max Tries

Agents now have a safety mechanism that tracks the number of times a tool is invoked during an execution session. If the agent exceeds this limit, execution is interrupted and an exception is thrown. By default the limit is 5 calls, and it count for each tool individually.&#x20;

You can customize this value with the `toolMaxTries()` method at agent level, or use `setMaxTries()` on the tool level. Setting max tries on single tool takes precedence over the global setting.

```php
try {

    $result = YouTubeAgent::make()
        ->toolMaxTries(5) // Max number of calls for each tool
        ->addTool(
            // Tool level config takes precedence over the global setting
            CustomTool::make()->setMaxTries(2)
        )
        ->chat(...);
        
} catch (ToolMaxTriesException $exception) {
    // do something
}
```

### Monitoring & Debugging

Neuron automatically manages the tool loop for you, based on what the LLM decided to call.

To watch inside this workflow you should connect your Agent to the [Inspector monitoring dashboard](https://inspector.dev/) in order to see the tool call execution flow in real-time.

After you sign up at the link above, make sure to set the `INSPECTOR_INGESTION_KEY` variable in the application environment file to start monitoring:

{% code title=".env" %}
```
INSPECTOR_INGESTION_KEY=nwse877auxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```
{% endcode %}

<figure><img src="../.gitbook/assets/tools-2.png" alt=""><figcaption></figcaption></figure>

In the image below you can see all the details about the execution of the tool to retrieve the transcription of the video:

<figure><img src="../.gitbook/assets/tools-3.png" alt=""><figcaption></figcaption></figure>

## Tool Properties

Neuron allows you to define the format of the data you want to receive into the tool function. You can nest these objects inside each other to define complex data structures.

### ToolProperty

This class represent a simple scalar value like string, int, or boolean.

```php
namespace App\Neuron\Tools;

use NeuronAI\Tools\PropertyType;
use NeuronAI\Tools\Tool;
use NeuronAI\Tools\ToolProperty;

class MyTool extends Tool
{
    public function __construct(){...}
	
    protected function properties(): array
    {
        return [
            new ToolProperty(
                name: 'arg',
                type: PropertyType::STRING,
                description: 'Describe the value you expect',
                required: true
            )
        ];
    }
    
    public function __invoke(string $arg){...}
}
```

### ArrayProperty

The `ArrayProperty` allows you to require a list of items with specific characteristics.

Use the argument `items` to specify the data type of the array elements. In the example below we ask for an array of string.

```php
namespace App\Neuron\Tools;

use NeuronAI\Tools\PropertyType;
use NeuronAI\Tools\Tool;
use NeuronAI\Tools\ArrayProperty;
use NeuronAI\Tools\ToolProperty;

class MyTool extends Tool
{
    public function __construct(){...}
	
    protected function properties(): array
    {
        return [
            new ArrayProperty(
                name: 'prop_array',
                description: 'Describe the value you expect',
                required: true,
                items: new ToolProperty(
                    name: 'prop',
                    type: PropertyType::STRING,
                    description: 'Describe the value you expect',
                    required: true
                )
            )
        ];
    }
    
    public function __invoke(string $arg){...}
}
```

#### Max and Min limits

The ArrayProperty allows you also to define limitations about the size of the expected array using `minItems` and `maxItems` arguments.

```php
$property = new ArrayProperty(
    name: "tags",
    description: "List of tags associated with the item",
    required: true,
    items: new ToolProperty(
        name: "tag",
        type: PropertyType::STRING,
        description: "A single tag",
        required: true
    ),
    minItems: 1,
    maxItems: 10
);
```

### ObjectProperty

Similar to the array example above you can define an object data structure:

```php
namespace App\Neuron\Tools;

use NeuronAI\Tools\PropertyType;
use NeuronAI\Tools\Tool;
use NeuronAI\Tools\ObjectProperty;
use NeuronAI\Tools\ToolProperty;

class MyTool extends Tool
{
    public function __construct(){...}
	
    protected function properties(): array
    {
        return [
            new ObjectProperty(
                name: 'colors',
                description: 'RGB color',
                required: true,
                properties: [
                    new ToolProperty(
                        name: 'r',
                        type: PropertyType::NUMBER,
                        description: 'The red part of the RGB',
                        required: true
                    ),
                    new ToolProperty(
                        name: 'g',
                        type: PropertyType::NUMBER,
                        description: 'The green part of the RGB',
                        required: true
                    ),
                    new ToolProperty(
                        name: 'b',
                        type: PropertyType::NUMBER,
                        description: 'The blue part of the RGB',
                        required: true
                    )
                ]
            )
        ];
    }
    
    public function __invoke(string $arg){...}
}
```

### Structured Tool Input

If the obect you want has many properties you can pass a structured PHP class to the `ObjectProperty` instead of defining the schema manually. Neuron will provide you with an instance of this class as the input argument of the tool function:

```php
namespace App\Neuron\Tools;

use App\Neuron\Dto\Color;
use NeuronAI\Tools\PropertyType;
use NeuronAI\Tools\Tool;
use NeuronAI\Tools\ToolProperty;

class MyTool extends Tool
{
    public function __construct(){...}
	
    protected function properties(): array
    {
        return [
            new ObjectProperty(
                name: 'color',
                description: 'Combination of colors',
                required: true,
                class: Color::class
            )
        ];
    }
    
    public function __invoke(Color $color){...}
}
```

Here is how the Colors class looks like:

```php
<?php

namespace App\Neuron\Dto;

use NeuronAI\StructuredOutput\SchemaProperty;

class Color
{
    #[SchemaProperty(description: "The RED part of the RGB", required: true)]
    public float $r;
    
    #[SchemaProperty(description: "The GREEN part of the RGB", required: true)]
    public float $g;
    
    #[SchemaProperty(description: "The BLUE part of the RGB", required: true)]
    public float $b;
}
```

## Provider Tools

Some providers offer the possibility to use their built-in tools like web\_search, file\_search, and others instead of relying on external services. Even they offer this service they introduce a lot of constraints using these tools. The most flexible and reliable way to add cpabailities to your agents remains the Tools and Toolkit systems.

You can add a provider tool as usual in the tools array of your agent:

```php
use NeuronAI\Tools\ProviderTool;

class MyAgent extends Agent
{
    public function provider(): AIProviderInterface
    {
        return new OpenAIResponses(
            key: 'OPENAI_API_KEY',
            model: 'OPENAI_MODEL',
        );
    }

    public function tools(): array
    {
        return [
            ProviderTool:make(
                type: 'web_search'
            )->setOptions([...]),
        ];
    }
}
```

Currently only [OpenAIResponses](ai-provider.md#openairesponses), [Gemini](ai-provider.md#gemini), and [Anthropic](ai-provider.md#anthropic) support these tools.

## Toolkits

The philosophy behind Neuron's toolkit system emerged from a fundamental observation during AI Agent Development: while individual tools provide specific capabilities, real-world AI agents often require coordinated sets of related functionalities.&#x20;

Rather than forcing developers to manually assemble collections of tools for common use cases, Neuron introduces toolkits as an abstraction layer that transforms how we think about agent capability composition.

The traditional approach requires instantiating each tool individually. Imagine you want to build agents that need mathematical reasoning – addition, subtraction, multiplication, division, and exponentiation tools must all be declared separately in the agent's tool configuration. This granular approach quickly becomes unwieldy when agents require comprehensive functionality sets.&#x20;

Toolkits represent Neuron's solution to this complexity, packaging tools created around the same scope into a single, coherent interface that can be attached to any agent with a single line of code.

Here is an example of the `CalculatorToolkit`:

```php
namespace NeuronAI\Tools\Toolkits\Calculator;

use NeuronAI\Tools\Toolkits\AbstractToolkit;

class CalculatorToolkit extends AbstractToolkit
{
    public function guidelines(): ?string
    {
        return "This toolkit allows you to perform mathematical operations. You can also use this functions to solve
        mathematical expressions executing smaller operations step by step to calculate the final result.";
    }

    public function provide(): array
    {
        return [
            SumTool::make(),
            SubtractTool::make(),
            MultiplyTool::make(),
            DivideTool::make(),
            ExponentiateTool::make(),
        ];
    }
}
```

The `AbstractToolkit` base class establishes a consistent interface that all toolkits inherit, ensuring predictable behavior across the framework.

The `guidelines()` method serves a particularly important function in agent development – it provides contextual information that helps the underlying language model understand not just what tools are available, but how they should be used together. In the case of the `CalculatorToolkit`, the guidelines explicitly suggest that complex mathematical expressions can be solved through step-by-step operations, guiding the agent toward effective problem-solving strategies.

The `provide()` method returns the array of tools included in the toolkit by default. When a toolkit is attached to an agent, the individual tools become available exactly as if they had been added separately, but without the cognitive overhead of managing multiple tool declarations. Here is how you can add it to your agent:

```php
<?php

namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\Tools\Calculator\CalculatorToolkit;

class MyAgent extends Agent
{
    ...
	
    public function tools(): array
    {
        return [
            CalculatorToolkit::make(),
        ];
    }
}
```

### Filters

During development of complex agents, I've frequently encountered scenarios where a toolkit provides mostly the right functionality but includes tools that could lead to undesired behavior in specific contexts, or just need to be restricted and configured individually.&#x20;

#### Exclude

The `exclude()` method addresses this challenge elegantly, allowing developers to attach comprehensive toolkits while maintaining fine-grained control over available capabilities. This becomes particularly useful when working with specialized agents that need specific capabilities but you want to reduce the probability of an agent mistake, and reduce tokens consumption.

```php
class MyAgent extends Agent
{
    ...
	
    public function tools(): array
    {
    	return [
            CalculatorToolkit::make()->exclude([
                DivideTool::class,
                ExponentiateTool::class,
                MultiplyTool::class,
            ]),
        ];
    }
}
```

The exclusion mechanism operates at the class level, using fully qualified class names to identify tools for removal.&#x20;

#### Only

In the same way you can also use the mthod `only()` to request a sub-set of the available tools in the toolkit.

```php
class MyAgent extends Agent
{
    ...
	
    public function tools(): array
    {
    	return [
            CalculatorToolkit::make()->only([
                StandardDeviationTool::class,
                MedianTool::class,
            ]),
        ];
    }
}
```

#### With

Following the same pattern you may need to retrieve an instance of a specific tool from the toolkit to change its settings. You can do this using the `with()` method. You can pass the fully qualified class name to declare what tool you want to retrieve, and the tool instance will be injected into the callback so you can change its settings and return it back.

```php
class MyAgent extends Agent
{
    ...
	
    public function tools(): array
    {
    	return [
            MySQLToolkit::make()
                ->with(
                    MySQLSchemaTool::class, 
                    fn (ToolInterface $tool) => $tool->setMaxTries(1)
                ),
        ];
    }
}
```

From an extensibility perspective, the toolkit system opens remarkable opportunities for community contribution and ecosystem growth. The consistent interface means that third-party developers can create domain-specific toolkits that integrate seamlessly with Neuron's architecture. A developer building agents for financial applications might create a FinancialToolkit that includes tools for currency conversion, interest calculation, and risk assessment. Similarly, a WebScrapingToolkit could package HTTP request tools, HTML parsing capabilities, and data extraction utilities into a single, reusable component.

## Available Toolkits

Neuron ships with several built-in tools and toolkits that allows you to quickly equip your agents with many skills. You can use these tools individually or attach entire toolkits with a single line of code.

### Calculator

The CalculatorToolkit provides a comprehensive suite of computational tools designed to make your AI agents performs accurate calculations. It can seamlessly integrates with complementary toolkits that provide data access—such as database connectors, CSV processors, API clients, or spreadsheet readers—enabling AI agents to perform sophisticated statistical calculations, and deliver comprehensive insights in response to complex business queries.

```php
<?php

namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\Tools\Toolkits\Calculator\CalculatorToolkit;

class MyAgent extends Agent
{
    ...
    
    protected function tools(): array
    {
        return [
            CalculatorToolkit::make(),
        ];
    }
}
```

<table data-header-hidden><thead><tr><th width="253"></th><th></th></tr></thead><tbody><tr><td>sum</td><td>NeuronAI\Tools\Toolkits\Calculator\SumTool</td></tr><tr><td>subtract</td><td>NeuronAI\Tools\Toolkits\Calculator\SubtractTool</td></tr><tr><td>multiply</td><td>NeuronAI\Tools\Toolkits\Calculator\MultiplyTool</td></tr><tr><td>divide</td><td>NeuronAI\Tools\Toolkits\Calculator\DivideTool</td></tr><tr><td>exponential</td><td>NeuronAI\Tools\Toolkits\Calculator\ExponentialTool</td></tr><tr><td>square root</td><td>NeuronAI\Tools\Toolkits\Calculator\SquareRootTool</td></tr><tr><td>nth root</td><td>NeuronAI\Tools\Toolkits\Calculator\NthRootTool</td></tr><tr><td>mean</td><td>NeuronAI\Tools\Toolkits\Calculator\MeanTool</td></tr><tr><td>median</td><td>NeuronAI\Tools\Toolkits\Calculator\MedianTool</td></tr><tr><td>mode</td><td>NeuronAI\Tools\Toolkits\Calculator\ModeTool</td></tr><tr><td>standard deviation</td><td>NeuronAI\Tools\Toolkits\Calculator\StandardDeviationTool</td></tr><tr><td>variance</td><td>NeuronAI\Tools\Toolkits\Calculator\VarianceTool</td></tr></tbody></table>

### Calendar

​This toolkit provides comprehensive date and time operations. Use these tools to make your agent able to work with dates, times, formatting, calculations, and timezone conversions.

```php
<?php

namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\Tools\Toolkits\CalendarToolkit\CalendarToolkit;

class MyAgent extends Agent
{
    ...
    
    protected function tools(): array
    {
        return [
            CalendarToolkit::make(),
        ];
    }
}
```

<table><thead><tr><th width="205"></th><th></th></tr></thead><tbody><tr><td>current_datetime</td><td>NeuronAI\Tools\Toolkits\Calendar\CurrentDateTimeTool</td></tr><tr><td>get_timestamp</td><td>NeuronAI\Tools\Toolkits\Calendar\GetTimestampTool</td></tr><tr><td>format_date</td><td>NeuronAI\Tools\Toolkits\Calendar\FormatDateTool</td></tr><tr><td>date_difference</td><td>NeuronAI\Tools\Toolkits\Calendar\DateDifferenceTool</td></tr><tr><td>add_time</td><td>NeuronAI\Tools\Toolkits\Calendar\AddTimeTool</td></tr><tr><td>subtract_time</td><td>NeuronAI\Tools\Toolkits\Calendar\SubtractTimeTool</td></tr><tr><td>calculate_age</td><td>NeuronAI\Tools\Toolkits\Calendar\CalculateAgeTool</td></tr><tr><td>convert_timezone</td><td>NeuronAI\Tools\Toolkits\Calendar\ConvertTimezoneTool</td></tr><tr><td>get_timezone_info</td><td>NeuronAI\Tools\Toolkits\Calendar\GetTimezoneInfoTool</td></tr><tr><td>get_weekday</td><td>NeuronAI\Tools\Toolkits\Calendar\GetWeekdayTool</td></tr><tr><td>is_weekend</td><td>NeuronAI\Tools\Toolkits\Calendar\IsWeekendTool</td></tr><tr><td>is_leap_year</td><td>NeuronAI\Tools\Toolkits\Calendar\IsLeapYearTool</td></tr><tr><td>get_days_in_month</td><td>NeuronAI\Tools\Toolkits\Calendar\GetDaysInMonthTool</td></tr><tr><td>start_of_period</td><td>NeuronAI\Tools\Toolkits\Calendar\StartOfPeriodTool</td></tr><tr><td>end_of_period</td><td>NeuronAI\Tools\Toolkits\Calendar\EndOfPeriodTool</td></tr><tr><td>get_week_number</td><td>NeuronAI\Tools\Toolkits\Calendar\GetWeekNumberTool</td></tr><tr><td>compare_dates</td><td>NeuronAI\Tools\Toolkits\Calendar\CompareDatesTool</td></tr><tr><td>is_date_in_range</td><td>NeuronAI\Tools\Toolkits\Calendar\IsDateInRangeTool</td></tr></tbody></table>

### MySQL & PostgreSQL

These toolkits make your agent able to interact with your database. If you ask "How many votes did the authors get in the last 14 days?", the agent doesn’t guess or hallucinate an answer. Instead, it recognizes that this question requires database access, identifies the appropriate tables involved and retrieves real data from your system.

<figure><img src="../.gitbook/assets/data-analyst-ai-agent-php-report.png" alt=""><figcaption></figcaption></figure>

All the tools in the MySQL and PostgreSQL toolkits require a [PDO](https://www.php.net/manual/en/class.pdo.php) instance as a constructor argument. If you are in a framework environment or you are already using an ORM in general, you can gather the underlying PDO instance from the ORM and pass it to the tools. You can learn more about this implementation strategy in this in-depth article: [https://inspector.dev/mysql-ai-toolkit-bringing-intelligence-to-your-database-layer-in-php/](https://inspector.dev/mysql-ai-toolkit-bringing-intelligence-to-your-database-layer-in-php/)

The PDO instance is basically a connection to a specific database, so you could aslo think to create dedicated credentials for your agent. It could be helpful to control the level of access your agent has to the database.&#x20;

Anyway you have separate tools for reading and writing to the database. If you are not confident about your agent behaviour you may not provide the writing tool.

```php
<?php

namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\Tools\Toolkits\MySQL\MySQLToolkit;
use NeuronAI\Tools\Toolkits\MySQL\PGSQLToolkit;

class MyAgent extends Agent
{
    ...
    
    protected function tools(): array
    {
        return [
            // Connect to a MySQL database
            MySQLToolkit::make(
                new \PDO("mysql:host=localhost;dbname=DB_NAME;charset=utf8mb4", "DB_USER", "DB_PASS"),
            ),
            
            // or Postgre database
            PGSQLToolkit::make(
                new \PDO("pgsql:host=localhost;dbname=DB_NAME;charset=utf8mb4", "DB_USER", "DB_PASS"),
            ),
        ];
    }
}
```

{% hint style="warning" %}
These examples refer to the `MySQLToolkit` but it's exactly the same using `PGSQLToolkit`.
{% endhint %}

#### MySQLSchemaTool / PGSQLSchemaTool

This tool allows agents to understand the structure of your database, enabling them to construct intelligent queries without requiring you to hardcode table structures or relationships into prompts. This tool essentially gives your agent the equivalent of a database administrator’s understanding of your schema, allowing it to craft queries that respect your data model and take advantage of existing indexes and relationships.

```php
namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\Tools\Toolkits\MySQL\MySQLSchemaTool;

class MyAgent extends Agent
{
    ...
    
    protected function tools(): array
    {
        return [
            MySQLSchemaTool::make(new \PDO(...)),
            
            // PGSQLSchemaTool::make(new \PDO(...)),
        ];
    }
}
```

This tool also accept a second argument `$tables`. You can basically pass a list of tables that you want to include in the schema information passed to the LLM. This is basically a way to limit the scope of the queries the agent will later execute on the database.

```php
namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\Tools\Toolkits\MySQL\MySQLSchemaTool;

class MyAgent extends Agent
{
    ...
    
    protected function tools(): array
    {
        return [
            MySQLSchemaTool::make(
                new \PDO(...),
                ['users', 'categories', 'articles', 'tags']
            ),
        ];
    }
}
```

By limiting the schema scope, you can create specialized agents that focus on specific areas of your application. A content management agent might only need access to articles, categories, and tags, while a user administration agent requires visibility into users, roles, and permissions tables. This approach not only improves performance but also reduces the cognitive load on the language model, leading to more accurate and focused responses.

#### MySQLSelectTool / PGSQLSelectTool

Use this tool to make your agent able to run SELECT query against the database.

```php
namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\Tools\Toolkits\MySQL\MySQLSchemaTool;
use NeuronAI\Tools\Toolkits\MySQL\MySQLSelectTool;

class MyAgent extends Agent
{
    ...
    
    protected function tools(): array
    {
        return [
            MySQLSchemaTool::make(new \PDO(...)),
            MySQLSelectTool::make(new \PDO(...)),
        ];
    }
}
```

#### MySQLWriteTool / PGSQLWriteTool

Use this tool to make your agent able to performs write operations against the database (INSERT, UPDATE, DELETE).

```php
namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\Tools\Toolkits\MySQL\MySQLSchemaTool;
use NeuronAI\Tools\Toolkits\MySQL\MySQLWriteTool;

class MyAgent extends Agent
{
    ...
    
    protected function tools(): array
    {
        return [
            MySQLSchemaTool::make(new \PDO(...)),
            MySQLWriteTool::make(new \PDO(...)),
        ];
    }
}
```

### Tavily&#x20;

This toolkit enable your agent to performs web search, page content extraction, and crawling.

```php
namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\Tools\Toolkits\Tavily\TavilyToolkit;

class MyAgent extends Agent
{
    ...
    
    protected function tools(): array
    {
        return [
            TavilyToolkit::make(
                key: 'TAVILY_API_KEY'
            ),
        ];
    }
}
```

#### Tavily Web Search

It makes your Agent able to search the web. It requires access to [Tavily APIs](https://tavily.com/).

```php
namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\Tools\Toolkits\Tavily\TavilySearchTool;

class MyAgent extends Agent
{
    ...
    
    protected function tools(): array
    {
        return [
            TavilySearchTool::make(
                key: 'TAVILY_API_KEY'
            ),
        ];
    }
}
```

You can customize the default options to retrieve search results by passing your preference in the `withOptions` method:

```php
TavilySearchTool::make(
    key: 'TAVILY_API_KEY'
)->withOptions([
    'days' => 30,
    'max_results' => 10,
]),
```

#### Tavily Extract

Extract web page content from an URL. It requires access to [Tavily APIs](https://tavily.com/).

```php
namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\Tools\Toolkits\Tavily\TavilyExtractTool;

class MyAgent extends Agent
{
    ...
    
    protected function tools(): array
    {
        return [
            TavilyExtractTool::make(
                key: 'TAVILY_API_KEY'
            ),
        ];
    }
}
```

#### Tavily Crawl

Tavily Crawl is a graph-based website traversal tool that can explore hundreds of paths in parallel with built-in extraction and intelligent discovery.

```php
namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\Tools\Toolkits\Tavily\TavilyCrawlTool;

class MyAgent extends Agent
{
    ...
    
    protected function tools(): array
    {
        return [
            TavilyCrawlTool::make(
                key: 'TAVILY_API_KEY'
            ),
        ];
    }
}
```

### Jina&#x20;

This toolkit enable your agent to performs web search, and read the content of a specific URL.

```php
namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\Tools\Toolkits\Jina\JinaToolkit;

class MyAgent extends Agent
{
    ...
    
    protected function tools(): array
    {
        return [
            JinaToolkit::make(
                key: 'JINA_API_KEY'
            ),
        ];
    }
}
```

#### Jina Web Search

It makes your Agent able to search the web. It requires access to [Jina API](https://jina.ai/).

```php
namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\Tools\Toolkits\Jina\JinaWebSearch;

class MyAgent extends Agent
{
    ...
    
    protected function tools(): array
    {
        return [
            JinaWebSearch::make(
                key: 'JINA_API_KEY'
            ),
        ];
    }
}
```

#### Jina URL Reader

Extract web page content from an URL. It requires access to [Jina API](https://jina.ai/).

```php
namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\Tools\Toolkits\Jina\JinaUrlReader;

class MyAgent extends Agent
{
    ...
    
    protected function tools(): array
    {
        return [
            JinaUrlReader::make(
                key: 'JINA_API_KEY'
            ),
        ];
    }
}
```

### Zep Memory

This toolkit connects a NeuronAI Agent to [Zep](https://www.getzep.com/) knowledge graph. This kind of system allows the agent to store relevant facts that may emerge during interactions with the agent over time. It's a long term memory in the sense that is not limited to the current conversation like the [ChatHistory](chat-history-and-memory.md) component does. It's an external persistent storage the agent will use to store and retrieve single pieces of information that can allow more personalized answers.

To learn more about the capabilities of these kind of system you can visit the Zep website: [https://www.getzep.com/](https://www.getzep.com/)

```php
namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\Tools\Toolkits\Zep\ZepLongTermMemoryToolkit;

class MyAgent extends Agent
{
    ...
    
    protected function tools(): array
    {
        return [
            ZepLongTermMemoryToolkit::make(
                key: 'ZEP_API_KEY',
                user_id: 'ID'
            ),
        ];
    }
}
```

The `user_id` arguments allows you to separate the long term memory in different silos if you want to serve multiple users. Based on your use case you can use this parameter as a "key" to separate the memory for the various entities the agent interact to (users, companies, etc.).

### AWS SES

#### Simple Email Service (SES)

This tool allows the agent to send an email message to one or more recipients, send notifications, confirmations, reports, or any other email-based communication. The tool handles proper email&#x20;delivery, and basic error handling automatically.

In order ti use this tool the AWS sdk for PHP must be installed.

```
composer require aws/aws-sdk-php
```

The tool gets an instance of the `SesClient` class from the AWS PHP sdk.

```php
namespace App\Neuron;

use Aws\Ses\SesClient;
use NeuronAI\Agent;
use NeuronAI\Tools\Toolkits\AWS\SESTool;

class MyAgent extends Agent
{
    ...
    
    protected function tools(): array
    {
        return [
            SESTool::make(
                sesClient: new SesCleint(...),
                fromEmail: 'my-address@email.com'
            ),
        ];
    }
}
```

### Supadata YouTube

This toolkit provides access to YouTube video transcriptions, metadata, channel information,\
and playlist data through Supadata.ai for content analysis and research purposes.

```php
namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\Tools\Toolkits\Supadata\SupadataYouTubeToolkit;

class MyAgent extends Agent
{
    ...
    
    protected function tools(): array
    {
        return [
            SupadataYouTubeToolkit::make(
                key: 'SUPADATA_API_KEY',
            ),
        ];
    }
}
```

#### Video Transcription

Allow the agent to retrieve the transcription of a youtube video.

```php
namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\Tools\Toolkits\Supadata\SupadataVideoTranscriptTool;

class MyAgent extends Agent
{
    ...
    
    protected function tools(): array
    {
        return [
            SupadataVideoTranscriptTool::make(
                key: 'SUPADATA_API_KEY',
            ),
        ];
    }
}
```

#### Video Metadata

Allow the agent to retrieve the metadata of a youtube video.

```php
namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\Tools\Toolkits\Supadata\SupadataVideoMetadataTool;

class MyAgent extends Agent
{
    ...
    
    protected function tools(): array
    {
        return [
            SupadataVideoMetadataTool::make(
                key: 'SUPADATA_API_KEY',
            ),
        ];
    }
}
```

#### Channel Metadata

Allow the agent to retrieve metadata from a YouTube channel including name, description, subscriber count, and more.

```php
namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\Tools\Toolkits\Supadata\SupadataYoutubeChannelTool;

class MyAgent extends Agent
{
    ...
    
    protected function tools(): array
    {
        return [
            SupadataYoutubeChannelTool::make(
                key: 'SUPADATA_API_KEY',
            ),
        ];
    }
}
```

#### Playlist Metadata

Allow the agent to retrieve metadata from a YouTube playlist including title, description, video count, and more.

```php
namespace App\Neuron;

use NeuronAI\Agent;
use NeuronAI\Tools\Toolkits\Supadata\SupadataYoutubePlaylistTool;

class MyAgent extends Agent
{
    ...
    
    protected function tools(): array
    {
        return [
            SupadataYoutubePlaylistTool::make(
                key: 'SUPADATA_API_KEY',
            ),
        ];
    }
}
```

## Parallel Tool Calls

If your agents are tool-hungry, you can enable parallel execution if the model ask for multiple tool calls in a single request.

#### Sequential Execution (Standard)

The agent calls tools **one at a time**, waiting for each to complete before starting the next:

```
1. Call tool A → wait for result
2. Call tool B → wait for result  
3. Call tool C → wait for result

Total time: Time(A) + Time(B) + Time(C)
```

#### Parallel Execution (With `pcntl`)

The agent calls **multiple tools simultaneously**, letting them run at the same time:

```
1. Call tool A, B, and C all at once
2. Wait for all to complete

Total time: Max(Time(A), Time(B), Time(C))
```

### Enable parallel execution

Just attach the `ParallelToolCalls` trait to your Agent or RAG agent:

```php
use NeuronAI\Tools\ParallelToolCalls;

class DemoAgent extends Agent
{
    use ParallelToolCalls;

    protected function provider(): AIProviderInterface
    {
        ...
    }

    protected function tools(): array
    {
        return [
            CalculatorToolkit::make(),
        ];
    }
}
```

### Requirements

To use this trait you need to install the `spatie/fork` package. For more information check out the GitHub repository: [https://github.com/spatie/fork](https://github.com/spatie/fork)

```
composer require spatie/fork
```

{% hint style="warning" %}
#### Limitations

This implementation requires the `pcntl` extension which is installed in many Unix and Mac systems by default.

**pcntl only works in CLI processes, not in a web context.**



If the `pcntl` extension is not present in the system running the agent (e.g. Windows machines) the trait automatically fallbacks to the standard tool calls execution. This can be helpful if you have a missmatch between your local development environment and the production environment. You can develop locally with `pcntl` disabled, then deploy to production environments where it may be enabled—**without modifying a single line of code**. The agent adapts automatically to whatever execution environment it finds itself in.
{% endhint %}

