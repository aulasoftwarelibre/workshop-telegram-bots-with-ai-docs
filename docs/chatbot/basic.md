# Basic Chatbot with AI

In this chapter, we will integrate AI-powered responses into a basic chatbot. The chatbot uses an AI model to generate replies based on user input.

## AI-Powered Responses

The chatbot relies on the `generateText` from Vercel SDK AI function to produce intelligent responses. This function takes two key inputs: the user’s message (prompt) and system instructions that define the AI’s role.

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

At this stage, the bot can respond intelligently using AI but lacks conversation memory to handle ongoing interactions.

## Full code

!!! example

    Update the next file to add the integration with the Vercel SDK AI

```ts title="src/lib/handlers/on-message.ts"
import { generateText } from 'ai'
import { Composer } from 'grammy'

import { registry } from '../ai/setup-registry'
import { environment } from '../environment.mjs'

export const onMessage = new Composer()

const PROMPT = `
You are a chatbot designed to help users book hair salon appointments for the next day.
`

onMessage.on('message:text', async (context) => {
  const userMessage = context.message.text

  // Generate the assistant's response using the conversation history
  const { text } = await generateText({
    model: registry.languageModel(environment.MODEL),
    prompt: userMessage,
    system: PROMPT,
  })

  // Reply with the generated text
  await context.reply(text)
})
```
