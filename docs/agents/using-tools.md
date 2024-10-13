# Displaying Available Time Slots

In this chapter, we’ll explore how our chatbot guides users to book a hair salon appointment for the next day by displaying the available time slots using Telegram buttons. We will focus on retrieving the available appointments and dynamically creating button options for the user.

## Showing Time Slot Buttons to Users

Our chatbot uses several components to display time slot buttons, allowing users to easily select their preferred appointment time. We’ll break down how this is achieved by combining the Telegram bot API, the Vercel AI SDK, and custom tools.

The chatbot leverages tools from the Vercel AI SDK to perform specific actions—like displaying buttons. In this case, the displaySelectionButtons tool is responsible for rendering Telegram buttons for available time slots. This tool works with the context provided by Telegram (via grammy), dynamically creating and sending button options to the user.

The chatbot uses the getFreeAppointments tool to retrieve available time slots for the next day. This tool queries the backend for any unreserved time slots. The slots are passed into the displaySelectionButtons tool, which then turns them into clickable options for the user to choose from.

The Telegram API, through grammy, allows the chatbot to interact with users in a conversational manner. Once the chatbot receives a message from the user, it generates a response using the AI model from Vercel, following the set rules in the prompt. If time slots are available, the bot uses InlineKeyboard to display them as buttons, providing a seamless way for users to select their appointment time.

To maintain a coherent flow, the chatbot uses the conversationRepository to log user messages and the bot’s responses. This ensures the chatbot can maintain context, such as remembering which time slots were offered, and guiding the user step-by-step toward confirming their appointment.

By combining these components, the chatbot provides a smooth and user-friendly booking experience. In the next chapter, we will cover how to handle the button clicks and confirm the reservation.

### Create the tool to display buttons

```ts title="src/lib/tools/display-selection-buttons.ts"
import { type CoreTool, tool } from "ai";
import { type Context, InlineKeyboard } from "grammy";
import { z } from "zod";

import { conversationRepository } from "../repositories/conversation";

export const buildDisplaySelectionButtons = (context: Context): CoreTool =>
  tool({
    description: "Use this tool to ask to the user with the available time slots as buttons they can select from.",
    execute: async ({ options, question }) => {
      console.log(`Called displaySelectionButtons tool with ${JSON.stringify({ options, question }, null, 2)}`);

      const buttonsRows = [];
      for (let index = 0; index < options.length; index += 2) {
        buttonsRows.push(
          options
            .slice(index, index + 2)
            .map((option: string) => ({
              data: `slot-${option}`,
              label: option,
            }))
            .map(({ data, label }) => InlineKeyboard.text(label, data))
        );
      }

      await context.reply(question, {
        reply_markup: InlineKeyboard.from(buttonsRows),
      });

      const content = `${question}: ${options.join(", ")}`;

      const chatId = context?.chat?.id as number;
      await conversationRepository.addMessage(chatId, "assistant", content);

      return null;
    },
    parameters: z.object({
      options: z.array(z.string()).describe("Array of time slots for the user to choose from"),
      question: z.string().describe("The question to ask the user"),
    }),
  });
```

### Add the tool and update the prompt

```ts title="src/lib/handlers/on-message.ts"
import { generateText } from 'ai'
import { Composer } from 'grammy'

import { registry } from '../ai/setup-registry'
import { environment } from '../environment.mjs'
import { conversationRepository } from '../repositories/conversation'
import { buildDisplaySelectionButtons } from '../tools/display-selection-buttons'
import { buildGetFreeAppointments } from '../tools/get-free-appointments'

export const onMessage = new Composer()

const PROMPT = `
You are a chatbot designed to help users book hair salon appointments for tomorrow.
You have access to several tools to help with this task and are restricted to handling appointment bookings only. You do not handle any other inquiries.
Your primary goal is to guide users through the process of booking their appointments efficiently.

You will follow these rules:

1. Appointment Search: Use the "getFreeAppointments" tool to search for available appointment times for tomorrow.
2. Display Options: After "getFreeAppointments" use always the "displaySelectionButtons" tool to ask the user the available time slots as buttons they can select from.
3. Confirm Appointment: After the user selects a time, use the confirmAppointment function to finalize their appointment request.
4. Single-purpose chatbot: You only help users book appointments for tomorrow and do not answer unrelated questions, you need to use tools to ask options.

If a user asks for information outside of these details, please respond with: "I'm sorry, but I cannot assist with that. For more information, please call us at (555) 456-7890 or email us at info@hairsalon.com."
`

onMessage.on('message:text', async (context) => {
  const userMessage = context.message.text
  const chatId = context.chat.id

  // Store the user's message
  await conversationRepository.addMessage(chatId, 'user', userMessage)

  // Retrieve past conversation history
  const messages = await conversationRepository.get(chatId)

  // Generate the assistant's response using the conversation history
  const { text } = await generateText({
    maxSteps: 2,
    messages,
    model: registry.languageModel(environment.MODEL),
    system: PROMPT,
    tools: {
      displaySelectionButtons: buildDisplaySelectionButtons(context),
      getFreeAppointments: buildGetFreeAppointments(),
    },
  })

  if (!text) {
    return
  }

  // Store the assistant's response
  await conversationRepository.addMessage(chatId, 'assistant', text)

  // Reply with the generated text
  await context.reply(text)
})
```

Now try to ask for an appointment and you will see how both tools are called in a row.