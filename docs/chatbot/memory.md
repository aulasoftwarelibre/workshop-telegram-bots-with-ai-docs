# Adding Memory to the Bot

In this chapter, we will add memory to our chatbot by storing messages exchanged between the bot and the users. This is useful when we want the bot to reference past interactions to provide a better conversational experience. We will use **Drizzle ORM** to manage our database schema and **PostgreSQL** as our database.

## Step 1: Creating the Messages Table

We will create a table named `messages` to store the conversations. 

### Example Table Structure

To create this table using **Drizzle ORM**, we can follow a similar structure to other tables created in the project. Below is a basic template:

```ts title="src/lib/db/schema/messages.ts"
import { bigint, jsonb, pgTable, serial, timestamp } from 'drizzle-orm/pg-core'

export const messages = pgTable('messages', {
  chatId: bigint({ mode: 'number' }).notNull(),
  content: jsonb('content').notNull(),
  messageId: serial('message_id').primaryKey(),
  occurredOn: timestamp('occurred_on').defaultNow().notNull(),
})
```

### Fields Breakdown

- **messageId**: Auto-incrementing primary key for each message.
- **chatId**: This stores the unique identifier of the chat session.
- **content**: Contains the structured JSON data of the message, typically formatted as a `CoreMessage` from the Vercel SDK.
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
      })
      .from(messages)
      .where(eq(messages.chatId, chatId))
      .orderBy(asc(messages.occurredOn))

    return result.map((row) => row.content as CoreMessage)
  }

  async addMessage(chatId: number, content: CoreMessage): Promise<void> {
    await database.insert(messages).values({
      chatId,
      content,
    })
  }

  async clear(chatId: number): Promise<void> {
    await database.delete(messages).where(eq(messages.chatId, chatId))
  }
}

export const conversationRepository = new ConversationRepository()
```


### Step 4: Implementing Memory in the Chatbot

In this step, we enhance the chatbot's functionality by adding memory capabilities, allowing it to remember past interactions with users. This enables the bot to provide a more personalized experience and improve its responses based on previous messages.


First we are going to reset the bot memory each time the `/start` command is executed, and add the welcome message.

```ts title="src/lib/commands/start.ts"
import type { CommandContext, Context } from 'grammy'

import { conversationRepository } from '../repositories/conversation'

export async function start(context: CommandContext<Context>): Promise<void> {
  const chatId = context.chat.id
  // Clear the conversation
  await conversationRepository.clear(chatId)

  const content = 'Welcome, how can I help you?'
  // Store the assistant's welcome message
  await conversationRepository.addMessage(chatId, {
    content,
    role: 'assistant',
  })

  await context.reply(content)
}
```

Now, we can add the user's messages and the bot's replies:

```ts title="src/lib/handlers/on-message.ts"
import { generateText } from 'ai'
import { Composer } from 'grammy'

import { registry } from '../ai/setup-registry'
import { environment } from '../environment.mjs'
import { conversationRepository } from '../repositories/conversation'

export const onMessage = new Composer()

const PROMPT = `
You are a chatbot designed to help users book hair salon appointments for the next day.
`

onMessage.on('message:text', async (context) => {
  const userMessage = context.message.text
  const chatId = context.chat.id

  // Store the user's message
  await conversationRepository.addMessage(chatId, {
    content: userMessage,
    role: 'user',
  })

  // Retrieve past conversation history
  const messages = await conversationRepository.get(chatId)

  // Generate the assistant's response using the conversation history
  const { responseMessages, text } = await generateText({
    messages,
    model: registry.languageModel(environment.MODEL),
    system: PROMPT,
  })

  // Store the assistant's response
  for await (const message of responseMessages) {
    await conversationRepository.addMessage(chatId, message)
  }

  // Reply with the generated text
  await context.reply(text)
})
```

By incorporating memory, the bot becomes more capable of engaging in meaningful dialogues, improving user satisfaction and the overall chat experience.
