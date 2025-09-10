# Repository Migration

We are moving Neuron to a dedicated GitHub organization to give the project its own **clear, independent identity**.&#x20;

Neuron GitHub organization: [**https://github.com/neuron-core**](https://github.com/neuron-core)

The new organization will also contain example repositories and other dedicated resources. Neuron remains **100% open source**, and this change makes it easier for the community to adopt, contribute, and grow together.

### **What you need to do (start from October 1st)**

* Open your project’s composer.json
* Find the dependency **inspector-apm/neuron-ai**
* Replace it with "**neuron-core/neuron-ai":**  "^2.0"
* Save the file and run your usual "composer update"

Here is how Neuron must be referenced in your `composer.json` file:

```json
"require": {
    ...
    "neuron-core/neuron-ai": "^2.0",
},
```

If you don’t update the package signature, you will no longer be able to receive updates **after October 1st**.

### Receive Updates

You can receive live updates subscribing to the Neuron newsletter: [**https://neuron-ai.dev**](https://neuron-ai.dev/)
