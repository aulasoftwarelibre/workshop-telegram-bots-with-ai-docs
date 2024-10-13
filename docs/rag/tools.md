# Dynamic Data Augmentation with Tools

In this section, we will enhance the chatbot’s functionality by introducing dynamic Data Augmentation through external tools. Unlike the static information used in earlier sections, this approach enables the bot to fetch real-time data and perform actions that respond to changing contexts, such as booking appointments or checking availability. This allows the bot to adapt to different scenarios dynamically, offering more relevant and personalized responses to users.

!!! warning

    For simplicity, we are using `chatId` instead of `userId`. Also, we are not handling concurrency or data integrity exceptions in this example.

## Step 1: Create the Appointments Table

This step involves defining the database schema for storing appointments, including a unique constraint to ensure that each time slot on a given day can only be reserved once. This schema serves as the foundation for the dynamic appointment scheduling system.


```ts title="src/lib/db/schema/appointments.ts"
import {
  bigint,
  date,
  pgTable,
  serial,
  time,
  uniqueIndex,
} from 'drizzle-orm/pg-core'

export const appointments = pgTable(
  'appointments',
  {
    chatId: bigint('chat_id', { mode: 'number' }),
    date: date('date', { mode: 'date' }).notNull(),
    id: serial('id').primaryKey(),
    timeSlot: time('time_slot').notNull(),
  },
  (table) => ({
    // Unique constraint on date and time slot
    uniqueDateTime: uniqueIndex('unique_date_time').on(
      table.date,
      table.timeSlot,
    ),
  }),
)
```

!!! note "Update the schema"

    Remember to generate and execute the migrations just we did in the [Adding Memory to the Bot](../chatbot/memory.md) section. 

## Step 2: Appointments Repository

The repository class contains methods for interacting with the database. It handles retrieving available appointments for a specified day and creating empty slots when none exist. The getFreeAppointmentsForDay method fetches all appointments for a given day and filters out the reserved ones (i.e., those with a chatId). If no appointments are found for the day, it dynamically creates them.

This logic ensures that the chatbot can always return relevant appointment data, even if none have been pre-created for that day. The dynamic nature of this repository is key to making the system respond to real-world conditions.

```ts title="src/lib/repositories/appointments.ts"
import { eq } from 'drizzle-orm/expressions'

import { db as database } from '../db/index'
import { appointments } from '../db/schema/appointments'

export class AppointmentRepository {
  async getFreeAppointmentsForDay(
    date: Date,
  ): Promise<Array<{ timeSlot: string }>> {
    const allAppointments = await database
      .select({
        chatId: appointments.chatId,
        timeSlot: appointments.timeSlot,
      })
      .from(appointments)
      .where(eq(appointments.date, date))

    // If no appointments exist for this day, create them
    if (allAppointments.length === 0) {
      await this.createEmptyAppointmentsForDay(date)
      return this.getFreeAppointmentsForDay(date)
    }

    // Filter free appointments where chatId is null (not reserved)
    const freeAppointments = allAppointments.filter(
      (appointment) => appointment.chatId === null,
    )

    return freeAppointments.map((appointment) => ({
      timeSlot: appointment.timeSlot,
    }))
  }

  private async createEmptyAppointmentsForDay(date: Date): Promise<void> {
    const openingTime = 9 // 9 AM
    const closingTime = 19 // 7 PM

    const timeSlots = Array.from(
      { length: closingTime - openingTime },
      (_, index) => {
        const time = `${(openingTime + index).toString().padStart(2, '0')}:00`
        return { date, timeSlot: time }
      },
    )

    // Insert empty appointments for each time slot
    await database.insert(appointments).values(timeSlots)
  }
}

export const appointmentsRepository = new AppointmentRepository()
```

### Step 3: Creating the tool

Here, we define a tool (getFreeAppointments) that fetches free appointments for the next day using the repository. The tool returns a markdown list of available time slots, which can be directly integrated into the chatbot’s responses. This tool encapsulates the repository logic, ensuring that the chatbot can retrieve dynamic appointment data without direct interaction with the database.

```ts title="src/lib/tools/get-free-appointments.ts"
import { type CoreTool, tool } from 'ai'
import { format } from 'date-fns'
import { z } from 'zod'

import { appointmentsRepository } from '../repositories/appointments'
import { tomorrow } from '../utils'

export const buildGetFreeAppointments = (): CoreTool =>
  tool({
    description:
      'Use this tool to search for available appointment times for tomorrow. Returns the response',
    execute: async () => {
      console.log(`Called getFreeAppointments tool`)

      const freeAppointments =
        await appointmentsRepository.getFreeAppointmentsForDay(tomorrow())

      if (freeAppointments.length === 0) {
        return `Sorry, there are no available appointments for tomorrow.`
      }

      const availableSlots = freeAppointments
        .map(
          (app) =>
            `- ${format(new Date(`1970-01-01T${app.timeSlot}`), 'HH:mm')}`,
        )
        .join('\n')

      return `Available appointments are:\n${availableSlots}.`
    },
    parameters: z.object({}),
  })
```

### Step 4: Adding the tool

Finally we incorporate the tool to the context of our bot.

```ts title="src/lib/handlers/on-message.ts"
import { generateText } from 'ai'
import { Composer } from 'grammy'

import { registry } from '../ai/setup-registry'
import { environment } from '../environment.mjs'
import { conversationRepository } from '../repositories/conversation'
import { buildGetFreeAppointments } from '../tools/get-free-appointments'

export const onMessage = new Composer()

const PROMPT = `
You are a chatbot designed to help users book hair salon appointments for the next day.
If the client ask for an appointment show the available slot times.

Use the tool 'getFreeAppointments' if you need to search the available appointments for tomorrow.

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
      getFreeAppointments: buildGetFreeAppointments(),
    },
  })

  // Store the assistant's response
  await conversationRepository.addMessage(chatId, 'assistant', text)

  // Reply with the generated text
  await context.reply(text)
})
```

By combining these tools and real-time data augmentation, the bot moves from being a static responder to a more interactive and context-aware assistant. This architecture allows for extending the bot with more tools in the future, enabling it to handle other types of dynamic data.
