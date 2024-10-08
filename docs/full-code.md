# Full code

## Registry

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

## Tools

```ts title="lib/tools.ts"
import type { CoreMessage } from 'ai'
import { Context, InlineKeyboard } from 'grammy'

import { generateEmbeddings } from './ai/embeddings'
import { db as database } from './db'
import { embeddings as embeddingsTable } from './db/schema/embeddings'
import {
  insertResourceSchema,
  type NewResourceParameters,
  resources,
} from './db/schema/resources'

export const createResource = async (
  input: NewResourceParameters,
): Promise<string> => {
  try {
    const { content } = insertResourceSchema.parse(input)

    const [resource] = await database
      .insert(resources)
      .values({ content })
      .returning()

    if (!resource) {
      return 'Resource not found'
    }

    const embeddings = await generateEmbeddings(content)
    await database.insert(embeddingsTable).values(
      embeddings.map((embedding) => ({
        resourceId: resource.id,
        ...embedding,
      })),
    )

    return 'Resource successfully created and embedded.'
  } catch (error) {
    return error instanceof Error && error.message.length > 0
      ? error.message
      : 'Error, please try again.'
  }
}

export const createConfirmAppointment = (availableTimeSlots: string[]) => {
  return async ({ slot }: { slot: string }): Promise<string> => {
    console.log(
      `Called createConfirmAppointment tool with ${JSON.stringify({ slot }, null, 2)}`,
    )

    const slotIndex = availableTimeSlots.indexOf(slot)
    if (slotIndex === -1) {
      return 'Sorry, that time slot is no longer available. Please choose another one.'
    }

    availableTimeSlots.splice(slotIndex, 1)

    return `Your appointment at ${slot} is confirmed!`
  }
}

export const createDisplaySelectionButtons = (
  context: Context,
  conversations: Map<number, CoreMessage[]>,
) => {
  return async ({
    options,
    question,
  }: {
    options: string[]
    question: string
  }): Promise<null> => {
    console.log(
      `Called createDisplaySelectionButtons tool with ${JSON.stringify({ options, question }, null, 2)}`,
    )

    const buttonsRows = []
    for (let index = 0; index < options.length; index += 2) {
      buttonsRows.push(
        options
          .slice(index, index + 2)
          .map((option: string) => ({ data: `slot-${option}`, label: option }))
          .map(({ data, label }) => InlineKeyboard.text(label, data)),
      )
    }

    await context.reply(question, {
      reply_markup: InlineKeyboard.from(buttonsRows),
    })

    const content = `${question}: ${options.join(', ')}`

    const chatId = context?.chat?.id as number
    const messages = conversations.get(chatId) ?? []
    messages.push({ content, role: 'assistant' })
    conversations.set(chatId, messages)

    return null
  }
}

export const createFindAvailableSlots = (availableTimeSlots: string[]) => {
  return async (): Promise<string[]> => {
    console.log(`Called createFindAvailableSlots tool`)
    return availableTimeSlots
  }
}
```

## Main bot

```ts title="src/main.ts"
import process from 'node:process'

import { type CoreMessage, generateText, tool } from 'ai'
import { Bot } from 'grammy'
import { z } from 'zod'

import { findRelevantContent } from './lib/ai/embeddings'
import { environment } from './lib/environment.mjs'
import {
  createConfirmAppointment,
  createDisplaySelectionButtons,
  createFindAvailableSlots,
} from './lib/tools'
import { createResource } from './lib/tools/resources'
import { registry } from './setup-registry'

const model = environment.MODEL

const PROMPT = `
You are a chatbot designed to help users book hair salon appointments for tomorrow. You have access to several tools to
help with this task and are restricted to handling appointment bookings only. You do not handle any other inquiries.
Your primary goal is to guide users through the process of booking their appointments efficiently.

You will follow these rules:

1. Appointment Search: Use the findAvailableSlots tool to search for available appointment times for tomorrow.
2. Display Options: Use the displaySelectionButtons tool to ask the user the available time slots as buttons they can select from.
3. Confirm Appointment: After the user selects a time, use the confirmAppointment function to finalize their appointment request.
4. Single-purpose chatbot: You only help users book appointments for tomorrow and do not answer unrelated questions, you need to use tools to ask options.

Proceed with these steps and maintain a helpful, friendly tone throughout the interaction. Make sure to validate user choices and guide them towards successfully booking an appointment.
`

const availableTimeSlots: string[] = ['09:00', '10:00', '11:00', '12:00']
const conversations: Map<number, CoreMessage[]> = new Map()

async function main(): Promise<void> {
  const bot = new Bot(environment.BOT_TOKEN)

  bot.command('start', async (context) => {
    const chatId = context.chatId
    const content = 'Welcome, how can I help you?'

    const messages: CoreMessage[] = []
    messages.push({ content, role: 'assistant' })
    conversations.set(chatId, messages)

    await context.reply(content)
  })

  bot.command('save', async (context) => {
    const content = context.match

    if (!content) {
      await context.reply('Not data found')
      return
    }

    const response = await createResource({ content })
    await context.reply(response)
  })

  bot.command('ask', async (context) => {
    const content = context.match

    if (!content) {
      await context.reply('No question found')
      return
    }

    const response = await generateText({
      maxSteps: 3,
      messages: [{ content, role: 'user' }],
      model: registry.languageModel(model),
      system: `You are a helpful assistant acting as the users' second brain.
      Use tools on every request.
      Be sure to getInformation from your knowledge base before answering any questions.
      if no relevant information is found in the tool calls, respond, "Sorry, I don't know."
      Be sure to adhere to any instructions in tool calls ie. if they say to responsd like "...", do exactly that.
      If the relevant information is not a direct match to the users prompt, you can be creative in deducing the answer.
      Keep responses short and concise. Answer in a single sentence where possible.
      If you are unsure, use the getInformation tool and you can use common sense to reason based on the information you do have.
      Use your abilities as a reasoning machine to answer questions based on the information you do have.
  `,
      tools: {
        getInformation: tool({
          description: `get information from your knowledge base to answer questions.`,
          execute: async ({ userQuestion }) =>
            findRelevantContent(userQuestion),
          parameters: z.object({
            similarQuestions: z
              .array(z.string())
              .describe('keywords to search'),
            userQuestion: z.string().describe('the users question'),
          }),
        }),
      },
    })

    if (!response.text) {
      await context.reply("Sorry, I don't know!")
      return
    }

    await context.reply(response.text)
  })

  bot.on('message:text', async (context) => {
    const chatId = context.chatId
    const userMessage = context.message.text

    const messages: CoreMessage[] = conversations.get(chatId) ?? []
    messages.push({ content: userMessage, role: 'user' })

    await bot.api.sendChatAction(chatId, 'typing')

    const { text } = await generateText({
      maxSteps: 2,
      messages,
      model: registry.languageModel(model),
      system: PROMPT,
      tools: {
        displaySelectionButtons: {
          description:
            'Use this tool to ask to the user with the available time slots as buttons they can select from.',
          execute: createDisplaySelectionButtons(context, conversations),
          parameters: z.object({
            options: z
              .array(z.string())
              .describe('Array of time slots for the user to choose from'),
            question: z.string().describe('The question to ask the user'),
          }),
        },
        findAvailableSlots: {
          description:
            'Use this tool to search for available appointment times for tomorrow. Returns an array of time slots.',
          execute: createFindAvailableSlots(availableTimeSlots),
          parameters: z.object({}),
        },
      },
    })

    if (text) {
      messages.push({ content: text, role: 'assistant' })
      await bot.api.sendMessage(chatId, text)
    }

    conversations.set(chatId, messages)
  })

  bot.callbackQuery(/slot-.+/, async (context) => {
    const chatId = context.chatId as number
    const slot = context.callbackQuery.data.slice(5)

    const messages = conversations.get(chatId) ?? []
    messages.push({ content: slot, role: 'user' })

    await bot.api.sendChatAction(chatId, 'typing')

    const { text } = await generateText({
      maxSteps: 2,
      messages,
      model: registry.languageModel(model),
      system: PROMPT,
      tools: {
        confirmAppointment: {
          description: 'Confirm the selected appointment.',
          execute: createConfirmAppointment(availableTimeSlots),
          parameters: z.object({
            slot: z
              .string()
              .describe('The selected time slot to confirm. Format HH:MM'),
          }),
        },
      },
    })

    if (text) {
      await bot.api.sendMessage(chatId, text)
      messages.push({ content: text, role: 'assistant' })
    }

    conversations.set(chatId, messages)
  })

  // Enable graceful stop
  process.once('SIGINT', () => bot.stop())
  process.once('SIGTERM', () => bot.stop())
  process.once('SIGUSR2', () => bot.stop())

  console.log('Bot started.')
  await bot.start()
}

main().catch((error) => console.error(error))
```
