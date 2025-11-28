# Installation

This guide will help you set up your local development environment for the Collab Digital Twins Platform.

## Prerequisites

Before you begin, ensure you have the following installed:

- **Node.js** (v18 or higher) — [Download here](https://nodejs.org/)
- **Yarn** or **npm** — Package manager for dependencies
- **Git** (optional, required for contributing) — [Download here](https://git-scm.com/)

Verify your installations:
```bash
node --version
yarn --version  # or: npm --version
git --version
```

## Clone the Repository
```bash
git clone https://github.com/collabdt/core.git
cd core
```

## Install Dependencies

Using Yarn (recommended):
```bash
yarn install
```

Or using npm:
```bash
npm install
```

## Environment Setup

Create a `.env` file in the project root:
```bash
cp .env.example .env
```

Update the environment variables as needed for your local setup.

## Start the Development Server
```bash
yarn dev
```

Or with npm:
```bash
npm run dev
```

The application should now be running at `http://localhost:3000` (or your configured port).

## Verify Installation

Open your browser and navigate to `http://localhost:3000`. You should see the CDT interface.

**Troubleshooting:** If you encounter issues, check our [GitHub Issues](https://github.com/collabdt/core/issues) or reach out via our [contact form](https://collabdt.org/home#contact).