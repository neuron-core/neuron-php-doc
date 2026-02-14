---
description: Managing errors fired by your agent.
---

# Error Handling

All exceptions fired from Neuron AI  are an extension of `NeuronException` . There are several types of exceptions that can help you understand what's going wrong, but because they inherit from the same root exception, they give you the ability to accurately detect agent errors in the context of your code:

```php
try {

    // Your code here...

} catch (NeuronAI\Exceptions\NeuronException $e) {
    // ...
} catch (NeuronAI\Exceptions\ProviderException $e) {
    // ...
} catch (NeuronAI\Exceptions\AgentException $e) {
    // ...
} catch (NeuronAI\Exceptions\ChatHistoryException $e) {
    // ...
} catch (NeuronAI\Exceptions\HttpException $e) {
    // ...
} catch (NeuronAI\Exceptions\ToolException $e) {
    // ...
} catch (NeuronAI\Exceptions\VectorStoreException $e) {
    // ...
} catch (NeuronAI\Exceptions\WorkflowException $e) {
    // ...
} catch (NeuronAI\Exceptions\DataReaderException $e) {
    // ...
}
```

### Monitoring & Debugging

If you want to be alerted on any error, consider to connect [**Inspector**](https://inspector.dev/) to your application.

After you sign up at the link above, make sure to set the `INSPECTOR_INGESTION_KEY` variable in the application environment file.

{% code title=".env" %}
```
INSPECTOR_INGESTION_KEY=nwse877auxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```
{% endcode %}

The Agent will automatically instrument itself. Learn more on the [documentation](observability.md) for other configuration options.

<figure><img src="../.gitbook/assets/inspector.png" alt=""><figcaption></figcaption></figure>
