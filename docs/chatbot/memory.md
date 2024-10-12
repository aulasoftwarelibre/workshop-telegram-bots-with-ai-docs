# Adding Memory to the Bot

In this chapter, we will add memory to our chatbot by storing messages exchanged between the bot and the users. This is useful when we want the bot to reference past interactions to provide a better conversational experience. We will use **Drizzle ORM** to manage our database schema and **PostgreSQL** as our database.

## Step 1: Creating the Messages Table

We will create a table named `messages` to store the conversations. 

### Example Table Structure

To create this table using **Drizzle ORM**, we can follow a similar structure to other tables created in the project. Below is a basic template:

```ts title="src/lib/db/schema/messages.ts"
import { bigint, pgTable, serial, text, varchar, timestamp } from 'drizzle-orm/pg-core'

export const messages = pgTable('messages', {
  chatId: bigint({ mode: 'number' }).notNull(),
  content: text('content').notNull(),
  messageId: serial('message_id').primaryKey(),
  occurredOn: timestamp('occurred_on').defaultNow().notNull(),
  role: varchar('role', { length: 50 }).$type<'user' | 'assistant' | 'system' | 'tool'>().notNull(),
})
```

### Fields Breakdown

- **messageId**: Auto-incrementing primary key for each message.
- **chatId**: This stores the unique identifier of the chat session.
- **role**: Defines whether the message is from the "user" or the "bot".
- **content**: The text content of the message.
- **occurredOn**: Automatically set timestamp for when the message was created.

!!! info

    For this project, this information is enough, but in other cases you may need to create a separate chat table or create another table for the user and store other data. 

## Step 2: Running Migrations

Once you have defined your table, you will need to generate the migration and run it in your PostgreSQL database.

### Generating and Applying the Migration

To generate the migration file, run:

    pnpm db:generate

This will generate the migration file based on your schema.

Next, run the migration to update your database schema:

    pnpm db:migrate

This will apply the migration and create the `messages` table in your database.

## Step 3: Storing and Retrieving Messages

Once the table is set up, you can integrate your bot to store messages in the `messages` table whenever a user sends a message or the bot replies. This will allow you to build memory features for your bot, improving its ability to reference past conversations. To do that we are going to create this repository class:

```ts title="src/lib/repositories/conversation.ts"
import type { CoreMessage } from 'ai'
import { asc, eq } from 'drizzle-orm/expressions'

import { db as database } from '../db/index'
import { messages } from '../db/schema/messages'

export class ConversationRepository {
  async get(chatId: number): Promise<CoreMessage[]> {
    const result = await database
      .select({
        content: messages.content,
        role: messages.role,
      })
      .from(messages)
      .where(eq(messages.chatId, chatId))
      .orderBy(asc(messages.occurredOn))

    return result.map(
      (row) =>
        ({
          content: row.content,
          role: row.role,
        }) as CoreMessage,
    )
  }

  async addMessage(
    chatId: number,
    role: CoreMessage['role'],
    content: string,
  ): Promise<void> {
    await database.insert(messages).values({
      chatId,
      content,
      role,
    })
  }

  async clear(chatId: number): Promise<void> {
    await database.delete(messages).where(eq(messages.chatId, chatId))
  }
}
```


### Step 4: Implementing Memory in the Chatbot

In this step, we enhance the chatbot's functionality by adding memory capabilities, allowing it to remember past interactions with users. This enables the bot to provide a more personalized experience and improve its responses based on previous messages.

By incorporating memory, the bot becomes more capable of engaging in meaningful dialogues, improving user satisfaction and the overall chat experience.

```ts
import process from 'node:process'

import { generateText } from 'ai'
import { Bot } from 'grammy'

import { environment } from './lib/environment.mjs'
import { ConversationRepository } from './lib/repositories/conversation'
import { registry } from './setup-registry'

const conversationRepository = new ConversationRepository()

async function main(): Promise<void> {
  const bot = new Bot(environment.BOT_TOKEN)

  bot.command('start', async (context) => {
    const chatId = context.chat.id
    // Clear the conversation
    await conversationRepository.clearConversation(chatId)

    const content = 'Welcome, how can I help you?'
    // Store the assistant's welcome message
    await conversationRepository.addMessage(chatId, 'assistant', content)

    await context.reply(content)
  })

  bot.on('message:text', async (context) => {
    const userMessage = context.message.text
    const chatId = context.chat.id

    // Store the user's message
    await conversationRepository.addMessage(chatId, 'user', userMessage)

    // Retrieve past conversation history
    const messages = await conversationRepository.get(chatId)

    // Generate the assistant's response using the conversation history
    const { text } = await generateText({
      messages,
      model: registry.languageModel(environment.MODEL),
      system:
        'You are a chatbot designed to help users book hair salon appointments for the next day.',
    })

    // Store the assistant's response
    await conversationRepository.addMessage(chatId, 'assistant', text)

    // Reply with the generated text
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
