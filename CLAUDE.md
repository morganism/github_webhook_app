# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a GitHub App that listens for and responds to webhook events from GitHub. It's built using Node.js with ES modules and the Octokit SDK. The app currently responds to pull request opened events by posting an automated comment.

## Development Commands

```bash
# Install dependencies
npm install

# Start the webhook server (listens on localhost:3000)
npm run server
```

## Environment Configuration

The application requires a `.env` file in the root directory with the following variables:

- `APP_ID` - The GitHub App ID
- `WEBHOOK_SECRET` - Secret for validating webhook signatures
- `PRIVATE_KEY_PATH` - Absolute path to the GitHub App's private key PEM file

## Architecture

### Single-File Application (app.js)

The entire application logic is contained in `app.js` using ES module syntax. Key components:

1. **App Initialization** (lines 22-28): Creates an Octokit App instance with authentication credentials from environment variables
2. **Webhook Event Handlers** (lines 34-53): Functions that process specific webhook events. The `handlePullRequestOpened` function uses the GitHub REST API to post comments on new PRs
3. **Event Registration** (line 56): Registers handlers for specific webhook events using `app.webhooks.on()`
4. **HTTP Server** (lines 70-88): Creates a Node.js HTTP server with middleware that validates webhook signatures, parses payloads, and routes events to the appropriate handlers

### Webhook Flow

1. GitHub sends webhook POST requests to `/api/webhook` endpoint
2. The middleware validates the webhook signature using `WEBHOOK_SECRET`
3. The payload is parsed and the event type is identified
4. The corresponding event handler is triggered
5. The handler uses Octokit to make authenticated GitHub API calls

### Adding New Event Handlers

To handle additional webhook events:
1. Create an async handler function that accepts `{octokit, payload}`
2. Register it with `app.webhooks.on("event_name", handlerFunction)`
3. Use `octokit.request()` for GitHub API interactions

### Local Development with smee-client

The `smee-client` package is included as a dev dependency for forwarding webhook events from GitHub to your local development server. This allows testing webhooks without deploying the app.

## Module System

The project uses ES modules (`"type": "commonjs"` in package.json is misleading - the actual code uses ES6 `import`/`export` syntax). All files should use `import` and `export` statements, not `require()`.
