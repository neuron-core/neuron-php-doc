---
description: Connect providers specialized in processing Audio to Text and vice-versa
---

# Audio

Usually pure AI Audio services don't support full agentic abilities like tools and conversation. So, you can use these components as stanalone services in an agentic workflow, or use them inside an Agent since they implement the `AIProviderInterface` interface. In this case you can benefit from the agentic workflow features like middleware and guardrails.

These component can be helpful for creating local voice assistants for hands-free interaction with models. The typical flow involves capturing audio, transcribing it to text with a separate Speech-To-Text (STT) service, sending that text to an agent for processing, and then using Text-to-Speech (TTS) to speak the response.

### As an Agent provider

```php
namespace App\Neuron;

use NeuronAI\Agent\Agent;
use NeuronAI\Chat\Messages\UserMessage;
use NeuronAI\Providers\AIProviderInterface;
use NeuronAI\Providers\OpenAI\Audio\OpenAITextToSpeech;

class MyAgent extends Agent
{
    protected function provider(): AIProviderInterface
    {
        return new OpenAITextToSpeech(
            key: 'OPENAI_API_KEY',
            model: 'gpt-4o-mini-tts',
            voice: 'alloy',
        );
    }
}

// Run the agent
$message = MyAgent::make()
    ->chat(new UserMessage("Hi!"))
    ->getMessage();

// Retrieve the audio part of the message (it's in base64 format)
$audioBase64 = $message->getAudio()->getContent();

// Save the audio file
file_put_contents(__DIR__.'/assets/speech.mp3', base64_decode($audioBase64));
```

### Direct use

```php
$provider = new OpenAITextToSpeech(
    key: 'OPENAI_API_KEY',
    model: 'gpt-4o-mini-tts',
    voice: 'alloy',
);

// Generate speech from text
$message = $provider->chat(new UserMessage("Hi, I'm the creator of Neuron AI framework!"));

// Retrieve the audio part of the message (it's in base64 format)
$audioBase64 = $message->getAudio()->getContent();

// Save the audio file
file_put_contents(__DIR__.'/assets/speech.mp3', base64_decode($audioBase64));
```

## OpenAI Audio

### Text-To-Speech

```php
use NeuronAI\Providers\OpenAI\Audio\OpenAITextToSpeech;

$provider = new OpenAITextToSpeech(
    key: 'OPENAI_API_KEY',
    model: 'gpt-4o-mini-tts',
    voice: 'alloy',
);

// Generate speech from text
$message = $provider->chat(new UserMessage("Hi, I'm the creator of Neuron AI framework!"));

// Retrieve the audio part of the message (it's in base64 format)
$audioBase64 = $message->getAudio();

// Save the audio file
file_put_contents(__DIR__.'/assets/speech.mp3', base64_decode($audioBase64));
```

### Speech-To-Text

```php
use NeuronAI\Providers\OpenAI\Audio\OpenAISpeechToText;

$provider = new OpenAISpeechToText(
    key: 'OPENAI_API_KEY',
    model: 'gpt-4o-transcribe',
);

// Transcript the audio
$message = $provider->chat(
    new UserMessage([
        new TextContent('This audio is about a math lesson. Take care of the technical words.'),
        new AudioContent(__DIR__ . '/assets/intro.mp3', SourceType::URL)
    ])
);

// Print the text gathered from the audio file
echo $message->getContent();
```

## ElevenLabs

### Text-To-Speech

```php
use NeuronAI\Providers\ElevenLabs\ElevenLabsTextToSpeech;

$provider = new ElevenLabsTextToSpeech(
    key: 'ELEVENLABS_API_KEY',
    model: 'eleven_multilingual_v2',
    voice: 'alloy',
);

// Generate speech from text
$message = $provider->chat(new UserMessage("Hi, I'm Valerio from Italy!"));

// Retrieve the audio part of the message (it's in base64 format)
$audioBase64 = $message->getAudio();

// Save the audio file
file_put_contents(__DIR__.'/asserts/speech.mp3', base64_decode($audioBase64));
```

### Speach-To-Text

```php
use NeuronAI\Providers\ElevenLabs\ElevenLabsSpeechToText;

$provider = new ElevenLabsSpeechToText(
    key: 'ELEVENLABS_API_KEY',
    model: 'scribe_v2',
);

// Transcript the audio
$message = $provider->chat(
    new UserMessage(
        new AudioContent(__DIR__ . '/assets/intro.mp3', SourceType::URL)
    )
);

// Print the text gathered from the audio file
echo $message->getContent();
```

## ZAI

### Speech-To-Text

```php
use NeuronAI\Providers\ZAI\Audio\ZAITranscription;

$provider = new ZAITranscription(
    key: 'ZAI_API_KEY',
    model: 'glm-asr-2512',
);

// Transcript the audio
$message = $provider->chat(
    new UserMessage(
        new AudioContent(__DIR__ . '/assets/intro.mp3', SourceType::URL)
    )
);

// Print the text gathered from the audio file
echo $message->getContent();
```
