# Running an Echo Bot

In this section, we will go through how to run a basic echo bot using the Grammy library. This bot listens for messages from users and replies with the same message.

## Setting up the Echo Bot

The following code sets up a simple bot that echoes back any text message it receives.

### Step 1: Import Required Modules

We start by importing the necessary modules. `dotenv` is used to load environment variables, and `Bot` comes from the **grammy** library, which is used to interact with Telegram bots:
    ```ts
    import process from 'node:process'
    import { Bot } from 'grammy'

    import { environment } from './lib/environment.mjs'
    ```

### Step 2: Initialize the Bot

In the `main` function, we initialize the bot using the token from the environment variable `BOT_TOKEN`. This token is essential for interacting with the Telegram API:
    ```ts
    async function main(): Promise<void> {
      const bot = new Bot(environment.BOT_TOKEN)
    ```

### Step 3: Add Commands

The bot is programmed to respond to a `/start` command with a welcome message. This is useful for onboarding users or providing initial instructions:
    ```ts
    bot.command('start', async (context) => {
      const content = 'Welcome, how can I help you?'
      await context.reply(content)
    })
    ```

### Step 4: Handle Messages

The core functionality of this echo bot is to listen for text messages. Whenever the bot receives a text message, it sends the same message back to the user:
    ```ts
    bot.on('message:text', async (context) => {
      const userMessage = context.message.text
      await context.reply(userMessage)
    })
    ```

### Step 5: Graceful Shutdown

To ensure that the bot stops gracefully when the application is terminated (for example, by `SIGINT` or `SIGTERM`), we add the following event listeners:
    ```ts
    process.once('SIGINT', () => bot.stop())
    process.once('SIGTERM', () => bot.stop())
    process.once('SIGUSR2', () => bot.stop())
    ```

### Step 6: Start the Bot

Finally, the bot is started with the `bot.start()` method:
    ```ts
    await bot.start()
    }

    main().catch((error) => console.error(error))
    ```

This code initializes the bot, listens for text messages, and echoes the received message back to the user. The bot will stop safely when the application receives termination signals.

## Running the Bot

To run the bot, follow these steps:

1. **Copy the environment configuration**:  
   Rename the provided `.env.example` file to `.env` to set up your environment variables. You can do this using the following command:
    ```bash
    cp .env.example .env
    ```

2. **Update the `.env` file**:  
   Open the `.env` file and set the required environment variables, especially the `BOT_TOKEN` with your Telegram bot token:
    ```bash
    BOT_TOKEN=your-telegram-bot-token
    ```

3. **Install dependencies**:  
   Make sure the required packages are installed by running:
    ```bash
    pnpm install
    ```

4. **Run the bot**:  
   Start the bot in development mode:
    ```bash
    pnpm run dev
    ```

After following these steps, your bot will be running, and you can start chatting with it. The bot will respond to the `/start` command with a welcome message and echo any text message you send.

## Full code

```ts title="src/main.ts"
import process from 'node:process'

import { environment } from './lib/environment.mjs'
import { Bot } from 'grammy'

dotenv.config()

async function main(): Promise<void> {
  const bot = new Bot(environment.BOT_TOKEN)

  bot.command('start', async (context) => {
    const content = 'Welcome, how can I help you?'

    await context.reply(content)
  })

  bot.on('message:text', async (context) => {
    const userMessage = context.message.text

    await context.reply(userMessage)
  })

  // Enable graceful stop
  process.once('SIGINT', () => bot.stop())
  process.once('SIGTERM', () => bot.stop())
  process.once('SIGUSR2', () => bot.stop())

  await bot.start()
}

main().catch((error) => console.error(error))
```