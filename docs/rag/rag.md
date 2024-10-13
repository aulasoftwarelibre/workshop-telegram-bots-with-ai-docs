# Retrieval-Augmented Generation

RAG (Retrieval-Augmented Generation) is a technique that combines a language model with a retrieval system to enhance the model’s responses. This method helps when the chatbot needs to provide specific information from external knowledge, such as documents or databases, that is not directly available in its training data. 

In our chatbot, we will use RAG to allow the bot to “learn” new information dynamically and then retrieve that information to answer user queries accurately. We’ll be integrating two commands into the bot: /learn and /ask.

## What is RAG?

RAG works by:

1. Retrieving relevant information from an external source, like a database or documents.
2. Augmenting a language model’s prompt with this retrieved information to generate more accurate and context-aware responses.

By splitting information into chunks and generating embeddings, we can compare the similarity of user queries with stored content. When a user asks a question, the system retrieves the most relevant content and uses it to improve the bot’s answer.

## What are Embeddings?

Embeddings are numerical representations of text that capture the meaning of words or sentences. In our case, embeddings allow us to represent pieces of content (e.g., text about services, pricing, or additional details) as vectors. By comparing the distance (e.g., cosine similarity) between these vectors and a user query’s embedding, we can determine how similar the stored content is to the user’s question.

## Creating the `/learn` command

The /learn command allows the chatbot to “learn” new content by generating embeddings for the input text and storing it in the database for later retrieval.

Here’s the code for `/learn`:

```ts title="src/lib/commands/learn.ts"
import type { CommandContext, Context } from 'grammy'

import { createResource } from '../ai/resources'

export async function learn(context: CommandContext<Context>): Promise<void> {
  const content = context.match

  if (!content) {
    await context.reply('No data found')
    return
  }

  const response = await createResource({ content })
  await context.reply(response)
}
```

## Creating the `/ask` command

```ts title="src/lib/commands/ask.ts"
import { generateText } from 'ai'
import type { CommandContext, Context } from 'grammy'

import { findRelevantContent } from '../ai/embeddings'
import { registry } from '../ai/setup-registry'
import { environment } from '../environment.mjs'

export async function ask(context: CommandContext<Context>): Promise<void> {
  const userQuery = context.match

  // Find relevant content using embeddings
  const relevantContent = await findRelevantContent(userQuery)

  if (relevantContent.length === 0) {
    await context.reply("Sorry, I couldn't find any relevant information.")
    return
  }

  // Generate the response with the RAG-enhanced prompt
  const { text } = await generateText({
    messages: [{ content: userQuery, role: 'user' }],
    model: registry.languageModel(environment.MODEL),
    // Combine the relevant content into the system prompt
    system: `
      You are a chatbot designed to help users book hair salon appointments.
      Here is some additional information relevant to your query:
  
      ${relevantContent.map((content) => content.name).join('\n')}
      
      Answer the user's question based on this information.
      If a user asks for information outside of these details, please respond with: "I'm sorry, but I cannot assist with that. For more information, please call us at (555) 456-7890 or email us at info@hairsalon.com."
    `,
  })

  // Reply with the generated text
  await context.reply(text)
}
```


## Adding the new commands

Remember to add these two new commands to the bot so that they can be used:

```ts title="src/main.ts"
import process from 'node:process'

import { Bot } from 'grammy'

import { ask } from './lib/commands/ask'
import { learn } from './lib/commands/learn'
import { start } from './lib/commands/start'
import { environment } from './lib/environment.mjs'
import { onMessage } from './lib/handlers/on-message'

async function main(): Promise<void> {
  const bot = new Bot(environment.BOT_TOKEN)

  bot.command('start', start)
  bot.command('learn', learn)
  bot.command('ask', ask)

  bot.use(onMessage)

  // Enable graceful stop
  process.once('SIGINT', () => bot.stop())
  process.once('SIGTERM', () => bot.stop())
  process.once('SIGUSR2', () => bot.stop())

  await bot.start()
}

main().catch((error) => console.error(error))
```


!!! info "How `createResource` and `findRelevantContent` works?"

    The code of both functions are extracted from [Vercel SDK AI RAG Guide](https://sdk.vercel.ai/docs/guides/rag-chatbot).
    You can find a more extended description there, but basically this is the flow:

	1.	Finding Relevant Content:
    The bot uses embeddings to compare the user’s query to stored content in the database. The method findRelevantContent searches for the most similar chunks using cosine similarity. If the similarity score is above a certain threshold (in this case, 0.3), the content is considered relevant.
    2.	Prompt Injection:
    Once the relevant content is found, it is combined into the PROMPT. This augmented prompt is passed into the generateText method, allowing the chatbot to provide an informed response based on the retrieved content.
    3.	Generate Text with Custom Prompt:
    The generateText method now includes the additional content in the system prompt. This augments the bot’s ability to respond in a contextually aware manner by incorporating specific information from the retrieved data.

With RAG, our bot can learn new information dynamically and retrieve relevant content to enhance its responses. By leveraging embeddings and prompt injection, the bot becomes more capable of answering user questions accurately. This setup demonstrates how RAG can be applied to improve interactions, making the bot more flexible and intelligent while still being grounded in specific data sources.


!!! exercise

    Add the information we had in the prompt:

	1.	`/learn Our salon offers a haircut service for $25.`
	2.	`/learn Our salon provides hair color services for $50.`
	3.	`/learn We also offer a manicure service for $15.`
	4.	`/learn Our opening hours are Monday to Saturday from 9 AM to 7 PM.`
	5.	`/learn Our salon is closed on Sundays.`

    And them does some questions:

    1. `/ask What are your opening hours?`
    2. `/ask How much is a haircut?`
    3. `/ask Say my name`
