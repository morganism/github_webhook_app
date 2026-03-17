# GitHub Webhook App

A Node.js application that responds to GitHub webhook events using the Octokit SDK. This app demonstrates how to build a GitHub App that listens for repository events and automatically performs actions in response.

## What This Does

Currently, this app:
- Listens for pull request opened events
- Automatically posts a welcome comment on newly opened pull requests
- Validates webhook signatures for security
- Provides a foundation for adding more webhook event handlers

## Table of Contents

- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Detailed Setup](#detailed-setup)
- [Local Development](#local-development)
- [Testing Webhooks](#testing-webhooks)
- [Extending the App](#extending-the-app)
- [Tips and Tricks](#tips-and-tricks)
- [Deployment](#deployment)
- [Troubleshooting](#troubleshooting)
- [Architecture](#architecture)

## Prerequisites

- **Node.js** (v18 or higher recommended)
- **npm** (comes with Node.js)
- **A GitHub account** with permissions to create GitHub Apps
- **ngrok** or **smee.io** (for local webhook testing)

## Quick Start

### 1. Create a GitHub App

1. Go to your GitHub account Settings → Developer settings → GitHub Apps
2. Click "New GitHub App"
3. Fill in the required fields:
   - **GitHub App name**: Choose a unique name (e.g., "My Webhook Responder")
   - **Homepage URL**: `http://localhost:3000` (for development)
   - **Webhook URL**: Use a smee.io channel or ngrok URL (see [Testing Webhooks](#testing-webhooks))
   - **Webhook secret**: Generate a random string (e.g., `openssl rand -hex 32`)
4. Under **Permissions**, grant:
   - **Repository permissions**:
     - Pull requests: Read & Write
     - Issues: Read & Write
5. Under **Subscribe to events**, check:
   - Pull request
6. Click "Create GitHub App"
7. On the app's page, click "Generate a private key" and download the `.pem` file

### 2. Install the App

1. From your GitHub App's page, click "Install App"
2. Choose the account/organization and repositories where you want to install it
3. Complete the installation

### 3. Configure the Environment

```bash
# Clone this repository
git clone https://github.com/morganism/github_webhook_app.git
cd github_webhook_app

# Install dependencies
npm install

# Create .env file
cat > .env << 'EOF'
APP_ID="your_app_id"
WEBHOOK_SECRET="your_webhook_secret"
PRIVATE_KEY_PATH="/path/to/your/private-key.pem"
EOF
```

Replace the values:
- `APP_ID`: Found on your GitHub App's settings page
- `WEBHOOK_SECRET`: The secret you created in step 1
- `PRIVATE_KEY_PATH`: Full absolute path to the `.pem` file you downloaded

### 4. Start the Server

```bash
npm run server
```

You should see:
```
Server is listening for events at: http://localhost:3000/api/webhook
Press Ctrl + C to quit.
```

### 5. Test It

1. Set up webhook forwarding (see [Testing Webhooks](#testing-webhooks))
2. Open a pull request in a repository where your app is installed
3. Watch the console logs and see the automated comment appear on the PR!

## Detailed Setup

### Understanding the Configuration

Your `.env` file contains three critical pieces of information:

1. **APP_ID**: Identifies your GitHub App to GitHub's API
2. **WEBHOOK_SECRET**: Used to verify that webhook payloads genuinely come from GitHub (prevents spoofing)
3. **PRIVATE_KEY_PATH**: Path to your app's private key, used for authenticating API requests

### Security Best Practices

- **Never commit your `.env` file** - it's already in `.gitignore`
- **Store the private key securely** - don't share it or commit it
- **Use a strong webhook secret** - at least 32 characters of random data
- **Rotate credentials if compromised** - generate a new private key from your GitHub App settings

## Local Development

### Using smee.io (Recommended for Development)

Smee.io is a webhook payload delivery service that forwards GitHub webhooks to your local machine.

1. **Create a Smee channel**:
   ```bash
   npx smee-client --url https://smee.io/new --path /api/webhook --port 3000
   ```

   Or visit https://smee.io/ and click "Start a new channel"

2. **Update your GitHub App's webhook URL** to the smee.io URL (e.g., `https://smee.io/abcd1234`)

3. **Start the smee client** (in a separate terminal):
   ```bash
   npx smee-client --url https://smee.io/your-channel-id --path /api/webhook --port 3000
   ```

4. **Start your app**:
   ```bash
   npm run server
   ```

Now webhooks will flow: GitHub → smee.io → your local server

### Using ngrok

Alternatively, use ngrok to create a public URL for your local server:

1. **Install ngrok**: https://ngrok.com/download

2. **Start your app**:
   ```bash
   npm run server
   ```

3. **Start ngrok** (in a separate terminal):
   ```bash
   ngrok http 3000
   ```

4. **Copy the HTTPS forwarding URL** (e.g., `https://abcd1234.ngrok.io`)

5. **Update your GitHub App's webhook URL** to `https://abcd1234.ngrok.io/api/webhook`

Note: ngrok URLs change each time you restart (unless you have a paid account).

## Testing Webhooks

### Manual Testing

1. Open a pull request in a repository where your app is installed
2. Watch your server console for logs
3. Check the PR for the automated comment

### Viewing Webhook Deliveries

1. Go to your GitHub App settings
2. Click "Advanced" in the sidebar
3. View "Recent Deliveries" to see webhook payloads and responses
4. Click "Redeliver" to resend a webhook (useful for debugging)

### Testing Specific Events

You can test different webhook events by:
- Creating issues
- Commenting on PRs
- Pushing commits
- Creating releases

Just make sure your app subscribes to those events and has the necessary permissions.

## Extending the App

### Adding New Event Handlers

The app is designed to be easily extensible. Here's how to add a new webhook handler:

#### Example: Respond to New Issues

```javascript
// Add this handler function in app.js
async function handleIssueOpened({octokit, payload}) {
  console.log(`Received an issue opened event for #${payload.issue.number}`);

  try {
    await octokit.request("POST /repos/{owner}/{repo}/issues/{issue_number}/comments", {
      owner: payload.repository.owner.login,
      repo: payload.repository.name,
      issue_number: payload.issue.number,
      body: "Thanks for opening this issue! We'll take a look soon.",
      headers: {
        "x-github-api-version": "2022-11-28",
      },
    });
  } catch (error) {
    console.error(`Error posting comment: ${error}`);
  }
}

// Register the handler
app.webhooks.on("issues.opened", handleIssueOpened);
```

#### Example: React to PR Reviews

```javascript
async function handlePullRequestReviewSubmitted({octokit, payload}) {
  const review = payload.review;
  const pr = payload.pull_request;

  console.log(`Review submitted on PR #${pr.number}: ${review.state}`);

  // Add a reaction to the review
  if (review.state === "approved") {
    await octokit.request("POST /repos/{owner}/{repo}/pulls/comments/{comment_id}/reactions", {
      owner: payload.repository.owner.login,
      repo: payload.repository.name,
      comment_id: review.id,
      content: "hooray",
      headers: {
        "x-github-api-version": "2022-11-28",
      },
    });
  }
}

app.webhooks.on("pull_request_review.submitted", handlePullRequestReviewSubmitted);
```

### Available Webhook Events

Common webhook events you can listen for:
- `issues.opened`, `issues.closed`, `issues.edited`
- `pull_request.opened`, `pull_request.closed`, `pull_request.synchronize`
- `pull_request_review.submitted`
- `push`
- `release.published`
- `issue_comment.created`
- `check_run.completed`
- `workflow_run.completed`

Full list: https://docs.github.com/en/webhooks/webhook-events-and-payloads

### Permissions Required

When adding new event handlers, ensure your GitHub App has the necessary permissions:

1. Go to your GitHub App settings
2. Scroll to "Permissions & events"
3. Update repository or organization permissions as needed
4. Users will need to accept the new permissions when they update the app

## Tips and Tricks

### Environment Variables

If you need to manage multiple environments (dev/staging/production), consider:

```bash
# .env.development
APP_ID="123456"
WEBHOOK_SECRET="dev-secret"
PRIVATE_KEY_PATH="/path/to/dev-key.pem"

# .env.production
APP_ID="789012"
WEBHOOK_SECRET="prod-secret"
PRIVATE_KEY_PATH="/path/to/prod-key.pem"
```

Then load the appropriate file:
```javascript
// In app.js
dotenv.config({ path: `.env.${process.env.NODE_ENV}` });
```

### Debugging Webhooks

Enable verbose logging:

```javascript
// Add this after creating the app instance
app.webhooks.on("*", async ({id, name, payload}) => {
  console.log(`Received webhook: ${name} (${id})`);
  console.log(JSON.stringify(payload, null, 2));
});
```

### Customizing Messages

Instead of hardcoding messages, use template strings:

```javascript
const messageForNewPRs = `
Thanks for opening a new PR, @${payload.pull_request.user.login}!

Please make sure you've:
- [ ] Updated documentation
- [ ] Added tests
- [ ] Followed our style guide

We'll review this soon!
`;
```

### Rate Limiting

GitHub API has rate limits. Check remaining quota:

```javascript
async function checkRateLimit(octokit) {
  const { data } = await octokit.request("GET /rate_limit");
  console.log(`Rate limit: ${data.rate.remaining}/${data.rate.limit}`);
  console.log(`Resets at: ${new Date(data.rate.reset * 1000)}`);
}
```

### Error Handling

Improve error handling with specific error types:

```javascript
try {
  await octokit.request(...);
} catch (error) {
  if (error.status === 403) {
    console.error("Permission denied - check app permissions");
  } else if (error.status === 404) {
    console.error("Resource not found");
  } else if (error.status === 422) {
    console.error("Validation failed:", error.response.data);
  } else {
    console.error("Unexpected error:", error);
  }
}
```

### Testing Locally Without GitHub

Create a test script to simulate webhooks:

```javascript
// test-webhook.js
import crypto from 'crypto';

const payload = JSON.stringify({
  action: 'opened',
  pull_request: { number: 123 },
  repository: { name: 'test-repo', owner: { login: 'testuser' } }
});

const signature = crypto
  .createHmac('sha256', process.env.WEBHOOK_SECRET)
  .update(payload)
  .digest('hex');

fetch('http://localhost:3000/api/webhook', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-GitHub-Event': 'pull_request',
    'X-Hub-Signature-256': `sha256=${signature}`
  },
  body: payload
});
```

### Conditional Logic

Add conditional logic to make your bot smarter:

```javascript
async function handlePullRequestOpened({octokit, payload}) {
  const pr = payload.pull_request;

  // Skip if PR is from a bot
  if (pr.user.type === "Bot") {
    return;
  }

  // Different message for first-time contributors
  const isFirstTimeContributor = pr.author_association === "FIRST_TIME_CONTRIBUTOR";
  const message = isFirstTimeContributor
    ? "Welcome! Thanks for your first contribution! 🎉"
    : "Thanks for the PR!";

  await octokit.request("POST /repos/{owner}/{repo}/issues/{issue_number}/comments", {
    owner: payload.repository.owner.login,
    repo: payload.repository.name,
    issue_number: pr.number,
    body: message,
    headers: { "x-github-api-version": "2022-11-28" },
  });
}
```

## Deployment

### Environment Considerations

When deploying to production:

1. **Change the host and port**:
   ```javascript
   const port = process.env.PORT || 3000;
   const host = process.env.HOST || '0.0.0.0';
   ```

2. **Update webhook URL** in your GitHub App settings to your production URL

3. **Use environment-specific credentials** (separate GitHub Apps for dev/prod)

### Deployment Platforms

This app can be deployed to:

- **Heroku**: Easy deployment, free tier available
  ```bash
  heroku create
  heroku config:set APP_ID=123456 WEBHOOK_SECRET=abc123
  # Upload private key as config var
  heroku config:set PRIVATE_KEY="$(cat private-key.pem)"
  git push heroku master
  ```

- **AWS Lambda**: Serverless option (requires modifications)
- **Docker**: Containerized deployment
- **Railway**: Simple deploy with GitHub integration
- **Vercel/Netlify**: Serverless functions (requires adapting code)

### Health Checks

Add a health check endpoint for monitoring:

```javascript
// Add before app.webhooks setup
app.octokit.request("GET /app").then(() => {
  console.log("✓ GitHub App authenticated successfully");
}).catch((error) => {
  console.error("✗ GitHub App authentication failed:", error);
  process.exit(1);
});
```

## Troubleshooting

### "Error! Status: 401" or "Error! Status: 403"

**Problem**: Authentication or permission issues

**Solutions**:
- Verify your `APP_ID` matches your GitHub App
- Check that `PRIVATE_KEY_PATH` points to the correct `.pem` file
- Ensure the private key file is readable (check permissions)
- Verify your app has the necessary permissions in GitHub settings
- Check that your app is installed on the repository you're testing with

### "Webhook signature verification failed"

**Problem**: Webhook secret mismatch

**Solutions**:
- Verify `WEBHOOK_SECRET` in `.env` matches GitHub App settings
- Check for extra spaces or newlines in the secret
- Regenerate the webhook secret if needed (update both GitHub and `.env`)

### No Webhooks Received

**Problem**: Webhooks not reaching your server

**Solutions**:
- Check that smee.io or ngrok is running
- Verify the webhook URL in GitHub App settings is correct
- Look at "Recent Deliveries" in GitHub App settings to see if webhooks are being sent
- Check for firewall or network issues blocking incoming connections
- Ensure your server is running on the expected port

### "Cannot find module" or Import Errors

**Problem**: Module resolution issues

**Solutions**:
- Run `npm install` to ensure all dependencies are installed
- Check that you're using Node.js v18+ (ES modules support)
- Verify `package.json` syntax is correct

### Webhook Times Out

**Problem**: Handler takes too long to respond

**Solutions**:
- Respond to the webhook quickly, then process asynchronously
- Consider using a queue (Redis, RabbitMQ) for long-running tasks
- GitHub expects a response within 10 seconds

### Rate Limited by GitHub API

**Problem**: Too many API requests

**Solutions**:
- Check your rate limit status: https://api.github.com/rate_limit
- Implement exponential backoff for retries
- Use conditional requests with `If-Modified-Since` headers
- Consider caching responses

## Architecture

### Component Overview

```
┌─────────────┐         ┌──────────────┐         ┌─────────────┐
│   GitHub    │ ──────> │  Webhook     │ ──────> │   Event     │
│             │ webhook │  Middleware  │ parsed  │   Handler   │
└─────────────┘         └──────────────┘         └─────────────┘
                              │                         │
                              │ validates               │
                              │ signature               │
                              v                         v
                        ┌──────────────┐         ┌─────────────┐
                        │   Webhook    │         │   GitHub    │
                        │    Secret    │         │     API     │
                        └──────────────┘         └─────────────┘
```

### Request Flow

1. GitHub sends a webhook POST request to `/api/webhook`
2. The request includes:
   - `X-Hub-Signature-256`: HMAC signature for verification
   - `X-GitHub-Event`: Event type (e.g., "pull_request")
   - `X-GitHub-Delivery`: Unique delivery ID
   - JSON payload with event details
3. `createNodeMiddleware` validates the signature using your webhook secret
4. If valid, the payload is parsed and routed to registered handlers
5. Your handler receives an authenticated `octokit` client and the payload
6. The handler makes GitHub API calls as needed
7. Response is sent back to GitHub

### File Structure

```
github_webhook_app/
├── app.js              # Main application code
├── package.json        # Dependencies and scripts
├── .env               # Configuration (not committed)
├── .gitignore         # Ignore patterns
├── bin/
│   └── post_install_gh.sh   # Post-install script
└── node_modules/      # Dependencies
```

## Resources

- **Octokit Documentation**: https://octokit.github.io/rest.js/
- **GitHub Webhooks Guide**: https://docs.github.com/en/webhooks
- **GitHub Apps Documentation**: https://docs.github.com/en/apps
- **Webhook Events Reference**: https://docs.github.com/en/webhooks/webhook-events-and-payloads
- **GitHub API Reference**: https://docs.github.com/en/rest

## License

ISC

## Contributing

Pull requests are welcome! For major changes, please open an issue first to discuss what you'd like to change.

## Support

For issues and questions:
- Open an issue: https://github.com/morganism/github_webhook_app/issues
- Check GitHub's webhook documentation
- Review recent webhook deliveries in your GitHub App settings
