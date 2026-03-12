---
description: Generate images from text
---

# Image

Usually pure AI Audio services don't support full agentic abilities like tools and conversation. So, you can use these components as stanalone services in an agentic workflow, or use them inside an Agent since they implement the `AIProviderInterface` interface. In this case you can benefit from the agentic workflow features like middleware and guardrails.

These component can be helpful for automating image generation based on textual prompts.

## Nano Banana (Google)

Google Gemini API provides a full multimodality experience, so you can just change the default model in your Gemini provider to generate images from prompts. Neuron also supports iteration on generated images with multi-turn conversations thanks to its multimodal message layer and chat history management.

Just configure one of the image generation model in Google Gemini provider:

```php
namespace App\Neuron;

use NeuronAI\Agent\Agent;
use NeuronAI\Chat\Messages\UserMessage;
use NeuronAI\Providers\AIProviderInterface;
use NeuronAI\Providers\Gemini\Gemini;

class MyAgent extends Agent
{
    protected function provider(): AIProviderInterface
    {
        return new Gemini(
            key: 'GEMINI_API_KEY',
            model: 'gemini-2.5-flash-image',
        );
    }
}

// Run the agent
$message = MyAgent::make()
    ->chat(new UserMessage("Generate an image of a venue hosting the best PHP conference!"))
    ->getMessage();

// Retrieve the image part of the message (it's in base64 format)
$imageBase64 = $message->getImage()->getContent();

// Save the image
file_put_contents(__DIR__.'/assets/cover.png', base64_decode($imageBase64));
```

## OpenAI Image

### As an Agent provider

```php
namespace App\Neuron;

use NeuronAI\Agent\Agent;
use NeuronAI\Chat\Messages\UserMessage;
use NeuronAI\Providers\AIProviderInterface;
use NeuronAI\Providers\OpenAI\Image\OpenAIImage;

class MyAgent extends Agent
{
    protected function provider(): AIProviderInterface
    {
        return new OpenAIImage(
            key: 'OPENAI_API_KEY',
            model: 'gpt-image-1.5',
        );
    }
}

// Run the agent
$message = MyAgent::make()
    ->chat(new UserMessage("Generate an image of a venue hosting the best PHP conference!"))
    ->getMessage();

// Retrieve the image part of the message (it's in base64 format)
$imageBase64 = $message->getImage()->getContent();

// Save the image
file_put_contents(__DIR__.'/assets/cover.png', base64_decode($imageBase64));
```

### Direct use

```php
use NeuronAI\Providers\OpenAI\Image\OpenAIImage;

$provider = new OpenAIImage(
    key: 'OPENAI_API_KEY',
    model: 'gpt-image-1.5',
);

// Generate speech from text
$message = $provider->chat(new UserMessage("Generate an image of a venue hosting the best PHP conference!"));

// Retrieve the image part of the message (it's in base64 format)
$imageBase64 = $message->getImage()->getContent();

// Save the image
file_put_contents(__DIR__.'/assets/cover.png', base64_decode($imageBase64));
```

## ZAI Image

### As an Agent Provider

```php
namespace App\Neuron;

use NeuronAI\Agent\Agent;
use NeuronAI\Chat\Messages\UserMessage;
use NeuronAI\Providers\AIProviderInterface;
use NeuronAI\Providers\ZAI\Image\ZAIImage;

class MyAgent extends Agent
{
    protected function provider(): AIProviderInterface
    {
        return new ZAIImage(
            key: 'ZAI_API_KEY',
            model: 'glm-image',
        );
    }
}

// Run the agent
$message = MyAgent::make()
    ->chat(new UserMessage("Generate an image of a venue hosting the best PHP conference!"))
    ->getMessage();

// Print the URL of the image
echo $message->getImage()->getContent();

```

### Direct Use

```php
use NeuronAI\Providers\ZAI\Image\ZAIImage;

$provider = new ZAIImage(
    key: 'ZAI_API_KEY',
    model: 'glm-image',
);

// Generate speech from text
$message = $provider->chat(new UserMessage("Generate an image of a venue hosting the best PHP conference!"));

// Print the URL of the image
echo $message->getImage()->getContent();
```
