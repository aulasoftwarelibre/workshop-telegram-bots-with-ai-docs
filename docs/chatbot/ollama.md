# Installing Ollama and Setting Up the Qwen2.5 Model

!!! info

    You can use free or commercial models. This tutorial includes both the possibility of using Ollama and OpenAI. You can add other providers if you wish, in the next chapter we explain how.

## What is Ollama?

Ollama is a platform that allows developers to run language models locally or on remote servers. It provides a seamless way to integrate different AI models into your applications, making it easier to deploy large language models without relying entirely on cloud services. Ollama supports multiple models, and you can interact with them via its API.

In this workshop, we are using Ollama as a provider to run the **Qwen2.5** language model for generating responses. This model is specified in our environment configuration and will be used alongside Telegram to power the chatbot's AI capabilities.

## How to Install Ollama

To install Ollama, follow these steps:

1. **Download Ollama**:  
   Visit the [Ollama website](https://ollama.com) and follow the instructions for your operating system (Linux, macOS, or Windows) to download the Ollama CLI (Command Line Interface).

2. **Install the CLI**:  
   After downloading the installer for your platform, run the installation process. For example, on Linux, you can install Ollama using:
        
        curl -fsSL https://ollama.com/install.sh | sh

3. **Verify Installation**:  
   To check that Ollama is successfully installed, open your terminal and run:
   
        ollama

   If the installation was successful, you should see a list of available commands and options for the Ollama CLI.

## Installing the Qwen2.5 Model

Once Ollama is installed, you need to download the `qwen2.5` model to use it in your project. Follow these steps:

1. **Pull the Model**:  
   Run the following command to download the Qwen2.5 model:

        ollama pull qwen2.5

2. **Verify the Model**:  
   After the model is downloaded, you can check that it's available by running:

        ollama list

   This will display a list of all the installed models, including `qwen2.5`.

