# Grizzly SaaS Monorepo

This repository is an NX monorepo that encompasses both frontend and backend applications, along with shared libraries and cron jobs. It leverages NestJS for the backend following Domain-Driven Design (DDD) and Clean Architecture principles, and Next.js for the frontend with Tailwind CSS and Framer Motion for styling and animations. Additionally, it integrates with AI services running on local servers.

## Table of Contents

- [Features](#features)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
  - [1. Clone the Repository](#1-clone-the-repository)
  - [2. Install Dependencies](#2-install-dependencies)
  - [3. Configure Environment Variables](#3-configure-environment-variables)
- [Running the Applications](#running-the-applications)
  - [1. Start the Backend (BFF)](#1-start-the-backend-bff)
  - [2. Start the Frontend](#2-start-the-frontend)
- [Cron Jobs](#cron-jobs)
- [Integrating with AI Services](#integrating-with-ai-services)
- [Building for Production](#building-for-production)
- [Testing](#testing)
- [Security Considerations](#security-considerations)
- [License](#license)
- [Acknowledgements](#acknowledgements)
- [Contact](#contact)

## Features

- **Backend (BFF)**: Built with NestJS following DDD and Clean Architecture.
- **Frontend**: Developed with Next.js, styled using Tailwind CSS, and enhanced with Framer Motion animations.
- **Cron Jobs**: Scheduled tasks managed within the backend application.
- **Shared Libraries**: Reusable code organized under the `libs/` directory.
- **AI Integration**: Communicates with AI services running on local servers.
- **Docker Support**: Each application includes a Dockerfile for containerization.
- **CI/CD**: GitHub Actions workflows are set up for continuous integration and deployment.

## Project Structure

```
grizzly-saas-monorepo/
├── apps/
│   ├── client/
│   │   ├── bff/          # Backend-for-Frontend (NestJS)
│   │   │   ├── src/
│   │   │   │   ├── app/
│   │   │   │   │   ├── app.controller.ts
│   │   │   │   │   ├── app.module.ts
│   │   │   │   │   └── cron.service.ts
│   │   │   ├── test/
│   │   │   ├── project.json
│   │   │   ├── tsconfig.app.json
│   │   │   └── ...
│   │   └── web/          # Frontend (Next.js)
│   │       ├── components/
│   │       │   └── AnimatedButton.jsx
│   │       ├── pages/
│   │       │   └── index.jsx
│   │       ├── public/
│   │       ├── styles/
│   │       │   └── globals.css
│   │       ├── tailwind.config.js
│   │       ├── next.config.js
│   │       ├── tsconfig.json
│   │       ├── project.json
│   │       └── ...
├── libs/
│   ├── domain/
│   │   └── user/
│   ├── infrastructure/
│   │   └── adapters/
│   └── client/
│       └── login/
│           ├── use-case/
│           └── presentation/
│               ├── react/
│               └── nest/
├── tools/
├── nx.json
├── package.json
├── tsconfig.base.json
└── workspace.json
```

## Prerequisites

Ensure you have the following installed on your machine:

- **Node.js (>=14.15.0)**: [Download Here](https://nodejs.org/)
- **npm** or **Yarn**: Preferred package manager.
- **Git**: Version control system.
- **Docker**: For containerizing applications (optional but recommended).
- **NX CLI**: Install globally to access NX commands.

  ```bash
  npm install -g nx
  ```

## Installation

### 1. Clone the Repository

```bash
git clone https://github.com/your_username/grizzly-saas-monorepo.git
cd grizzly-saas-monorepo
```

### 2. Install Dependencies

Install the dependencies for the entire monorepo:

```bash
npm install
```

*Alternatively, if you use Yarn:*

```bash
yarn install
```

### 3. Configure Environment Variables

Create a `.env` file in `apps/client/bff/` to store environment variables. Ensure this file is **not** committed to version control by verifying it's listed in `.gitignore`.

```env
# apps/client/bff/.env

HUGGINGFACE_API_KEY=your_hugging_face_api_token_here
AI_SERVICE_URL=http://localhost:8002/generate
```

**Notes:**

- **HUGGINGFACE_API_KEY**: Your Hugging Face API access token.
- **AI_SERVICE_URL**: URL of your AI service (FastAPI).

## Running the Applications

### 1. Start the Backend (BFF)

Navigate to the root of the monorepo and run:

```bash
npx nx serve client-bff
```

**Details:**

- **Port**: NestJS runs on port `3333` by default.
- **CORS**: Configured to allow requests from `http://localhost:4200` (Frontend).

### 2. Start the Frontend

In another terminal, navigate to the root of the monorepo and run:

```bash
npx nx serve client-web
```

**Details:**

- **Port**: Next.js runs on `http://localhost:4200`.
- **Tailwind CSS and Framer Motion**: Already configured and ready to use.

## Cron Jobs

Cron jobs are configured within the backend (BFF) using the `@nestjs/schedule` module.

### Current Configuration

- **Daily Task**: Executes a request to the AI service to generate a daily news summary.

**Configuration File:**

```typescript
// apps/client/bff/src/app/cron.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';
import axios from 'axios';

@Injectable()
export class CronService {
  private readonly logger = new Logger(CronService.name);

  // Runs daily at midnight
  @Cron(CronExpression.EVERY_DAY_AT_MIDNIGHT)
  async handleDailyTask() {
    this.logger.debug('Executing daily Llama task...');
    try {
      const response = await axios.post(process.env.AI_SERVICE_URL, {
        messages: [
          { role: 'system', content: 'You are a data aggregator.' },
          { role: 'user', content: 'Summarize today’s news.' }
        ],
        max_new_tokens: 200
      });
      this.logger.debug('Daily summary:', response.data.generated_text);
      // Add additional logic here, such as storing in a database or sending notifications
    } catch (error) {
      this.logger.error('Error executing daily Llama task:', error.message);
    }
  }
}
```

## Integrating with AI Services

Your frontend communicates with the Backend-for-Frontend (BFF), which in turn communicates with AI services (FastAPI).

### Communication Flow

1. **Frontend**: Sends a POST request to `/api/llama/generate`.
2. **Backend (BFF)**: Receives the request and forwards it to the AI service at `http://localhost:8002/generate`.
3. **AI Service (FastAPI)**: Processes the request and returns the response to the BFF.
4. **Backend (BFF)**: Returns the response to the frontend.
5. **Frontend**: Displays the response to the user.

### Example Request from Frontend

```javascript
// apps/client/web/pages/index.jsx

const handleGenerate = async () => {
  try {
    const response = await fetch('/api/llama/generate', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        messages: [
          { role: 'system', content: 'You are a helpful assistant.' },
          { role: 'user', content: input },
        ],
        max_new_tokens: 100,
      }),
    });

    const data = await response.json();
    setOutput(data.generated_text);
  } catch (error) {
    console.error('Error generating text:', error);
  }
};
```

## Building for Production

### 1. Build the Backend

```bash
npx nx build client-bff
```

### 2. Build the Frontend

```bash
npx nx build client-web
```

### 3. Dockerization (Optional)

Each application includes a `Dockerfile`. To build and run Docker containers:

1. **Backend (BFF)**

   ```bash
   cd apps/client/bff
   docker build -t grizzly-saas-bff:latest .
   docker run -d -p 3333:3333 grizzly-saas-bff:latest
   ```

2. **Frontend**

   ```bash
   cd apps/client/web
   docker build -t grizzly-saas-frontend:latest .
   docker run -d -p 4200:4200 grizzly-saas-frontend:latest
   ```

**Note:** Adjust ports and configurations as needed.

## Testing

### Run Tests for the Backend

```bash
npx nx test client-bff
```

### Run Tests for the Frontend

```bash
npx nx test client-web
```

## Security Considerations

- **Protect Your API Token:** Do not expose your `.env` file or API token in public repositories. Ensure that `.env` is listed in `.gitignore` to prevent accidental commits.
  
- **Docker Permissions:** Running Docker commands with `sudo` is common but consider configuring Docker to run without `sudo` for convenience and security.

- **Data Privacy:** Ensure that any data processed by the service complies with relevant data privacy laws and guidelines.

- **Sanitization de Entradas:** Implement measures to sanitize and validate inputs received from the frontend to prevent injections and other attacks.

## License

This project is licensed under the [MIT License](LICENSE).

## Acknowledgements

- [NX](https://nx.dev/) - Extensible Dev Tools for Monorepos.
- [NestJS](https://nestjs.com/) - A progressive Node.js framework for building efficient and scalable server-side applications.
- [Next.js](https://nextjs.org/) - The React Framework for Production.
- [Tailwind CSS](https://tailwindcss.com/) - A utility-first CSS framework.
- [Framer Motion](https://www.framer.com/motion/) - A production-ready motion library for React.
- [Hugging Face](https://huggingface.co/) - For providing the Llama models and the Transformers library.
- [Docker](https://www.docker.com/) - For containerizing applications.

## Contact

For any questions or support, please open an issue in the repository or contact the maintainer at [polribasrovira@gmail.com](mailto:polribasrovira@gmail.com).
