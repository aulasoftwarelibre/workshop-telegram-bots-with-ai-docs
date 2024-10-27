# Handling User Selection

In this chapter, we’ll focus on how the chatbot processes user selections of time slots and finalizes their booking by reserving the appointment in the database. We’ll examine how the user’s button action triggers the confirmation tool and how the appointment is stored and managed.

## Processing User Selections and Confirming the Appointment

Once the user has chosen a time slot using the button interface, the chatbot must handle this selection and confirm the booking. This process involves handling the button action, reserving the time slot, and providing feedback to the user.

### Managing Appointment Data

The AppointmentRepository is responsible for handling appointment data within our chatbot. It ensures that available time slots are fetched, new ones are created if necessary, and appointments are reserved when a user confirms a time slot.

The `reserve` is a new function than attempts to reserve an available slot. It updates the database, associating the selected time slot with the user’s chat ID. If successful, the slot is marked as reserved, and no other users can select it.


```ts title="src/lib/repositories/appointments.ts"
import { and, eq, isNull } from "drizzle-orm/expressions";

import { db as database } from "../db/index";
import { appointments } from "../db/schema/appointments";

export class AppointmentRepository {
  async getFreeAppointmentsForDay(date: Date): Promise<Array<{ timeSlot: string }>> {
    const allAppointments = await database
      .select({
        chatId: appointments.chatId,
        timeSlot: appointments.timeSlot,
      })
      .from(appointments)
      .where(eq(appointments.date, date));

    // If no appointments exist for this day, create them
    if (allAppointments.length === 0) {
      await this.createEmptyAppointmentsForDay(date);
      return this.getFreeAppointmentsForDay(date);
    }

    // Filter free appointments where chatId is null (not reserved)
    const freeAppointments = allAppointments.filter((appointment) => appointment.chatId === null);

    return freeAppointments.map((appointment) => ({
      timeSlot: appointment.timeSlot,
    }));
  }

  private async createEmptyAppointmentsForDay(date: Date): Promise<void> {
    const openingTime = 9; // 9 AM
    const closingTime = 19; // 7 PM

    const timeSlots = Array.from({ length: closingTime - openingTime }, (_, index) => {
      const time = `${(openingTime + index).toString().padStart(2, "0")}:00`;
      return { date, timeSlot: time };
    });

    // Insert empty appointments for each time slot
    await database.insert(appointments).values(timeSlots);
  }

  async reserve(chatId: number, date: Date, timeSlot: string): Promise<boolean> {
    const updated = await database
      .update(appointments)
      .set({ chatId })
      .where(and(eq(appointments.date, date), eq(appointments.timeSlot, timeSlot), isNull(appointments.chatId)))
      .returning()
      .execute();

    return updated.length > 0;
  }
}

export const appointmentsRepository = new AppointmentRepository();
```

### Confirming the Appointment

The `confirmAppointment` tool is used to handle the reservation once the user selects a time slot. The tool checks if the selected time slot is still available in the database. If the slot is free, it reserves it for the user by associating it with their chat ID. If the slot is already taken, it notifies the user that they need to pick a different time. This tool ensures that only available slots can be reserved, and it provides feedback based on the reservation’s success or failure.

```ts title="src/lib/tools/confirm-appointment.ts"
import { type CoreTool, tool } from "ai";
import type { Context } from "grammy";
import { z } from "zod";

import { appointmentsRepository } from "../repositories/appointments";
import { tomorrow } from "../utils";

export const buildConfirmAppointment = (context: Context): CoreTool =>
  tool({
    description: "Confirm the selected appointment.",
    execute: async ({ slot }) => {
      console.log(`Called confirmAppointment tool with ${JSON.stringify({ slot }, null, 2)}`);

      const chatId = context.chatId;
      if (!chatId) {
        return "Sorry, I cannot do the reserve.";
      }

      const isReserved = appointmentsRepository.reserve(chatId, tomorrow(), slot);

      if (!isReserved) {
        return "Sorry, that time slot is no longer available. Please choose another one.";
      }

      return `Your appointment at ${slot} is confirmed!`;
    },
    parameters: z.object({
      slot: z.string().describe("The selected time slot to confirm. Format HH:MM"),
    }),
  });
```

## Telegram Callback Listener for Button Actions

The onAppointment listener is responsible for capturing the user’s button presses and processing them within the ongoing conversation. When a user clicks a time slot button, the listener extracts the selected slot from the button’s callback data and integrates it into the conversation.


```ts title="src/lib/handlers/on-appointment.ts"
import { generateText } from 'ai'
import { Composer } from 'grammy'

import { registry } from '../ai/setup-registry'
import { environment } from '../environment.mjs'
import { conversationRepository } from '../repositories/conversation'
import { buildConfirmAppointment } from '../tools/confirm-appointment'

export const onAppointment = new Composer()

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

onAppointment.callbackQuery(/slot-.+/, async (context) => {
  const chatId = context.chatId as number
  const slot = context.callbackQuery.data.slice(5)

  await conversationRepository.addMessage(chatId, {
    content: slot,
    role: 'user',
  })
  const messages = await conversationRepository.get(chatId)

  await context.api.sendChatAction(chatId, 'typing')

  const { responseMessages, text } = await generateText({
    maxSteps: 2,
    messages,
    model: registry.languageModel(environment.MODEL),
    system: PROMPT,
    tools: {
      confirmAppointment: buildConfirmAppointment(context),
    },
  })

  // Store the assistant's response
  for await (const message of responseMessages) {
    await conversationRepository.addMessage(chatId, message)
  }

  if (!text) {
    return
  }

  // Reply with the generated text
  await context.reply(text)
})
```

And we need include this listener into the main bot file:

```ts title="src/main.ts"
import process from 'node:process'

import { Bot } from 'grammy'

import { ask } from './lib/commands/ask'
import { learn } from './lib/commands/learn'
import { start } from './lib/commands/start'
import { environment } from './lib/environment.mjs'
import { onAppointment } from './lib/handlers/on-appointment'
import { onMessage } from './lib/handlers/on-message'

async function main(): Promise<void> {
  const bot = new Bot(environment.BOT_TOKEN)

  bot.command('start', start)
  bot.command('learn', learn)
  bot.command('ask', ask)

  bot.use(onMessage)
  bot.use(onAppointment)

  // Enable graceful stop
  process.once('SIGINT', () => bot.stop())
  process.once('SIGTERM', () => bot.stop())
  process.once('SIGUSR2', () => bot.stop())

  await bot.start()
}

main().catch((error) => console.error(error))
```