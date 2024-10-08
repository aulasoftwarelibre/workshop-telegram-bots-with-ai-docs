# Workshop Telegram Bots with AI Template

This repository serves as a template for creating Telegram bots that leverage AI. You can quickly set up a development environment using GitHub Codespaces or run it locally using Docker.

## Getting Started

### Using This Template

To get started with the project, you can create your own copy of this template repository. Follow these steps:

1. Go to the [workshop-telegram-bots-with-ai-template](https://github.com/aulasoftwarelibre/workshop-telegram-bots-with-ai-template) repository.
2. Click on the **Use this template** button located at the top right corner of the page.
3. Select your account or organization and provide a name for your new repository.
4. Click **Create repository from template**.

### Setting Up with GitHub Codespaces

This repository is configured with a devcontainer, which allows you to easily develop in a consistent environment. To use GitHub Codespaces:

1. Once you have created your repository, navigate to it.
2. Click on the **Code** button and then select **Open with Codespaces**.
3. Click on **New codespace** to launch your development environment.

The devcontainer configuration will automatically run `pnpm install` to install the necessary dependencies.

### Running Locally

If you prefer to run the project locally, you'll need to set up Docker and Visual Studio Code (VSCode) with the necessary extensions.

#### Prerequisites

- **Docker**: Make sure Docker is installed and running on your machine. You can download it from [Docker's official website](https://www.docker.com/get-started).
- **Visual Studio Code**: Download and install VSCode from [Visual Studio Code's official website](https://code.visualstudio.com/).
- **Dev Containers Extension**: Install the [Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) for VSCode.

#### Steps to Run Locally

1. Clone your repository to your local machine:
   ```bash
   git clone https://github.com/your-username/your-repository-name.git
   cd your-repository-name
   ```
2.	Open the cloned repository in VSCode.
3.	When prompted, select Reopen in Container. This will build the Docker container defined in the .devcontainer folder.
