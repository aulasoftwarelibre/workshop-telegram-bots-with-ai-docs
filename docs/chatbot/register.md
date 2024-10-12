# Creating Registries in Vercel SDK

In this section, you will learn how to create registries for AI models in Vercel SDK. This setup allows you to register multiple AI providers and language models to be used in your project.

!!! info

    This step is usually not necessary, if you are only going to use one AI provider you can use it directly instead of creating a record.

## Setting up the Registry

The following steps will guide you on how to register AI providers such as OpenAI and Ollama using the Vercel SDK.

### Step 1: Import Required Modules

First, import the necessary modules from the AI SDK and other providers:

```ts
import { openai as originalOpenAI } from '@ai-sdk/openai'
import {
    experimental_createProviderRegistry as createProviderRegistry,
    experimental_customProvider as customProvider,
} from 'ai'
import { ollama as originalOllama } from 'ollama-ai-provider'
```

### Step 2: Create Custom Providers

Define custom providers for the AI models. For instance, we are using **Ollama** with a specific language model (`qwen-2_5`) and **OpenAI** with a custom model (`gpt-4o-mini`). Here is how you can define them:

```ts
const ollama = customProvider({
    fallbackProvider: originalOllama,
    languageModels: {
    'qwen-2_5': originalOllama('qwen2.5'),
    },
})

export const openai = customProvider({
    fallbackProvider: originalOpenAI,
    languageModels: {
    'gpt-4o-mini': originalOpenAI('gpt-4o-mini', {
        structuredOutputs: true,
    }),
    },
})
```

### Step 3: Create the Registry

Once the providers are defined, create the registry that will include these custom providers:

```ts
export const registry = createProviderRegistry({
    ollama,
    openai,
})
```

The `registry` now holds both **Ollama** and **OpenAI** providers, each registered with its specific language models.

### Step 4: Use the Registry

You can now use the `registry` in your project to manage and switch between AI providers and models as needed.

This completes the setup for registering AI providers in the Vercel SDK. You can expand this registry by adding more providers or models as required by your application.

Now, to use one or the other, edit the .env file and configure which provider and which model you want to use. Remember that if you want to use OpenAI you will have to have a Token API.

!!! warning

    It is possible that the free models do not work as well as the proprietary ones in the examples that use tools. Especially if they are small, since it is normal that in local we cannot run models with more than 12B of parameters. After the publication of this tutorial new and better open models may appear, try other options to see if they work better. If not you can always try a commercial model.


## Full code

```ts title="src/setup-registry.ts"
import { openai as originalOpenAI } from '@ai-sdk/openai'
import {
  experimental_createProviderRegistry as createProviderRegistry,
  experimental_customProvider as customProvider,
} from 'ai'
import { ollama as originalOllama } from 'ollama-ai-provider'

const ollama = customProvider({
  fallbackProvider: originalOllama,
  languageModels: {
    'qwen-2_5': originalOllama('qwen2.5'),
  },
})

export const openai = customProvider({
  fallbackProvider: originalOpenAI,
  languageModels: {
    'gpt-4o-mini': originalOpenAI('gpt-4o-mini', {
      structuredOutputs: true,
    }),
  },
})

export const registry = createProviderRegistry({
  ollama,
  openai,
})
```