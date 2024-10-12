# Basic Chatbot with AI

In this chapter, we will explore how to integrate an AI model into a basic chatbot.

## AI-Powered Responses

The main functionality of this chatbot is to use an AI model to generate responses based on user input. This is done using the `generateText` function, which takes a prompt (the user's message) and a system instruction to guide the AI's behavior.

### Understanding the Code

When a user sends a message to the chatbot, the following code generates a response using the AI model:

```ts
const { text } = await generateText({
    model: registry.languageModel(environment.MODEL),
    prompt: userMessage,
    system:
    'You are a chatbot designed to help users book hair salon appointments for the next day.',
})
```

### Parameters

1. **model**:
   This line retrieves the AI model specified in the environment configuration (`environment.MODEL`). The `registry.languageModel` function is used to select the appropriate model from the registry, which contains providers like OpenAI or Ollama. In this case, the model is used to generate responses based on the user’s input.

2. **prompt**:
   The `prompt` is the actual input message that the user has sent to the bot. It will be passed to the AI model so that it can generate an appropriate response. In this case, the user's message is stored in the `userMessage` variable.

3. **system**:
   The `system` field provides instructions to the AI model about how it should behave. This message ensures that the chatbot behaves like a virtual assistant specifically designed to help users book hair salon appointments. This is a way of giving the model context for its responses.

### How It Works

1. The user sends a message to the chatbot.
2. The `generateText` function takes the user’s message (`prompt`) and uses the model specified by `environment.MODEL`.
3. The system prompt ensures that the AI model knows it is a chatbot for booking appointments, guiding it to provide relevant responses.
4. The AI model processes the input and generates a response, which is then sent back to the user.

Now our bot answers using AI, but it has no memory and is not able to continue a conversation.

## Full code


```ts title="src/main.ts"
import process from 'node:process'

import { generateText } from 'ai'
import { Bot } from 'grammy'

import { environment } from './lib/environment.mjs'
import { registry } from './setup-registry'

async function main(): Promise<void> {
  const bot = new Bot(environment.BOT_TOKEN)

  bot.command('start', async (context) => {
    const content = 'Welcome, how can I help you?'

    await context.reply(content)
  })

  bot.on('message:text', async (context) => {
    const userMessage = context.message.text

    const { text } = await generateText({
      model: registry.languageModel(environment.MODEL),
      prompt: userMessage,
      system:
        'You are a chatbot designed to help users book hair salon appointments for the next day.',
    })

    await context.reply(text)
  })

  // Enable graceful stop
  process.once('SIGINT', () => bot.stop())
  process.once('SIGTERM', () => bot.stop())
  process.once('SIGUSR2', () => bot.stop())

  await bot.start()
}

main().catch((error) => console.error(error))
```
