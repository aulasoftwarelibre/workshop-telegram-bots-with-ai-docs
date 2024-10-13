# Data Augmentation Basics

Data augmentation is a technique used to enhance the performance of machine learning models by providing them with additional, relevant information. In the context of chatbots, particularly for specialized applications like booking hair salon appointments, data augmentation can significantly improve the quality of responses by enriching the context provided to the model.

## Implementing Basic Data Augmentation

To effectively implement data augmentation in our chatbot, we will utilize the `system` parameter in the `generateText` function. This parameter allows us to provide contextual information about the assistant's role and the relevant details about the business it represents.

For our hair salon chatbot, we can define a `PROMPT` variable that includes both the assistant's role and essential details about the salon. Here's how you can set it up just updating the system prompt:

```ts
const PROMPT = `
You are a chatbot designed to help users book hair salon appointments for the next day.
Here is some basic information about the salon:
- Services:
  - Haircut: $25
  - Hair Color: $50
  - Manicure: $15
- Opening Hours: Monday to Saturday, 9 AM to 7 PM
- Closing Day: Sunday

If a user asks for information outside of these details, please respond with: "I'm sorry, but I cannot assist with that. For more information, please call us at (555) 456-7890 or email us at info@hairsalon.example.com."
`;

const { text } = await generateText({
  messages,
  model: registry.languageModel(environment.MODEL),
  system: PROMPT,
});
```

By enriching the model’s understanding with specific information about the salon, we can ensure that it provides accurate answers related to services and availability. This is a straightforward method of implementing data augmentation, enhancing the user experience by making interactions more informative and context-aware.


!!! exercise "Example questions for the Chatbot"

    You can try some of these questions to see how the bot responds now that you have provided it with more information:

    - What services do you offer?
    - What are your opening hours?
    - How much is a haircut?
    - Can I get a pedicure?
    - What days are you open?
    - Do you have any special promotions?
    - Can I book an appointment for next week?
