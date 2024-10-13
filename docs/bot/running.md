# Running an Echo Bot

In this section, we will go through how to run a basic echo bot using the Grammy library. This bot listens for messages from users and replies with the same message.

## Setting up the Echo Bot

The following code sets up a simple bot that echoes back any text message it receives.

### Step 1: Add a command

The first step involves creating a simple `/start` command. This command responds with a welcome message to introduce the bot and let users know it’s ready to help. It’s a good practice to include a starting point like this for bots to guide the user experience, especially for first-time users.

```ts title="src/lib/commands/start.ts"
import type { CommandContext, Context } from 'grammy'

export async function start(context: CommandContext<Context>): Promise<void> {
  const content = 'Welcome, how can I help you?'
  await context.reply(content)
}
```

### Step 2: Handle Messages

This is where the echo functionality comes into play. The bot listens for incoming text messages from users and, upon receiving a message, responds by sending back the same message. This demonstrates the basic capability of the bot to handle and reply to user input.

```ts title="src/lib/handlers/on-message.ts"
import { generateText } from 'ai'
import { Composer } from 'grammy'

import { environment } from '../environment.mjs'

export const onMessage = new Composer()

onMessage.on('message:text', async (context) => {
  const userMessage = context.message.text
  await context.reply(userMessage)
})
```

### Step 3: Construct the bot

This section walks through setting up the core bot functionality. First, we initialize the bot using the token from environment variables, which is required for Telegram to authenticate and interact with your bot.

Next, we attach the `/start` command and the echo message handler (`onMessage`). The bot listens for incoming text and command events and processes them accordingly.

Additionally, we ensure the bot shuts down properly when the process receives termination signals, such as SIGINT or SIGTERM, making the bot more robust and production-ready.


```ts title="src/main.ts"
import process from 'node:process'

import { environment } from './lib/environment.mjs'
import { Bot } from 'grammy'

import { start } from './lib/commands/start'
import { environment } from './lib/environment.mjs'
import { onMessage } from './lib/handlers/on-message'


async function main(): Promise<void> {
  const bot = new Bot(environment.BOT_TOKEN)

  bot.command('start', start)

  bot.use(onMessage)

  // Enable graceful stop
  process.once('SIGINT', () => bot.stop())
  process.once('SIGTERM', () => bot.stop())
  process.once('SIGUSR2', () => bot.stop())

  await bot.start()
}

main().catch((error) => console.error(error))
```

## Running the Bot

This final section provides a step-by-step guide on how to set up and run the bot. It includes copying environment variables, updating the configuration with the correct Telegram bot token, installing the required dependencies, and finally running the bot in development mode.

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
