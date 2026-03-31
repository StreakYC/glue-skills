---
name: glue
description: Use when building, debugging, or deploying Glue automations with the Glue CLI and Glue runtime.
license: MIT
metadata:
  author: Streak
  version: "1.0"
---

# Glue

## Overview

Glue lets you write a single TypeScript file that responds to external events and takes actions. Glue handles the plumbing around webhook setup, trigger registration, account auth, execution history, and cloud hosting so your code can stay focused on business logic.

A typical Glue workflow is:

1. Sign in with `glue login`
2. Create a new Glue with `glue create`
3. Run it locally with `glue dev path/to/file.ts`
4. Trigger real events and iterate quickly
5. Deploy with `glue deploy path/to/file.ts`
6. Debug production behavior with `glue logs`, `glue describe`, and `glue replay`

## When to Use Glue

Use Glue when you need to:

- React to real events from services like GitHub, Gmail, Slack, Stripe, Intercom, webhooks, cron, Google Drive, Google Sheets, or Streak
- Write automation logic in a single TypeScript entrypoint instead of building webhook infrastructure yourself
- Debug event-driven behavior locally using real tunneled events
- Replay production events locally or on a deployed Glue to reproduce bugs
- Call third-party APIs with official SDKs using credentials managed by Glue accounts

## Scope Boundaries

This skill applies specifically to Glue CLI workflows and the Glue runtime.

- If the user is asking about generic TypeScript architecture outside Glue, answer directly using normal TypeScript patterns.
- If the user is asking about an unrelated deployment system, webhook framework, or job runner, do not force Glue concepts into the answer.
- If the user is extending Glue itself by adding new event sources or account providers, follow the connector conventions from the Glue backend repos instead of inventing new integration patterns.

## Getting Started

### Install the CLI

```bash
curl -fsSL https://glue.wtf/install.sh | bash
```

Or install directly with Deno:

```bash
deno install --unstable-kv --unstable-temporal -Agfrn glue jsr:@streak-glue/cli
```

### Sign in

```bash
glue login
glue whoami
```

### Create a new Glue

```bash
glue create
```

### Run locally

```bash
glue dev path/to/your-glue.ts
```

### Deploy

```bash
glue deploy path/to/your-glue.ts
```

## Core Concepts

### A Glue is a single TypeScript file

A Glue is just one entrypoint file. Import the runtime and register handlers at the top level.

```typescript
import { glue } from "jsr:@streak-glue/runtime";

glue.webhook.onPost((event) => {
  console.log("Webhook received", event.bodyText);
});
```

CRITICAL: Register handlers at the top level during initialization. Do not register handlers dynamically after the program has already started.

### Triggers vs actions

Glue handlers usually do two things:

- Register a trigger that listens for an external event
- Use credentials or SDK clients to take an action in response

For example, a Slack message or GitHub pull request can trigger your Glue, and then your code can call another API using the correct account credentials.

### Accounts and credential fetchers

Glue stores external service credentials as accounts. Your code should use Glue's runtime integrations and credential fetchers instead of hardcoding secrets.

CRITICAL: Never hardcode API keys, OAuth tokens, or account credentials in the Glue file or in checked-in `.env` files.

### Executions and replay

Every run of a deployed Glue creates an execution record with logs, status, and input data. Use replay tooling when debugging unexpected behavior.

```bash
glue replay <executionId>
```

To replay a production execution locally:

```bash
glue dev --replay <executionId> path/to/your-glue.ts
```

While `glue dev` is running, press `r` to replay the last event you received.

## Common Patterns

### Webhook Glue

```typescript
import { glue } from "jsr:@streak-glue/runtime";

glue.webhook.onGet((_event) => {
  console.log("GET request received");
});

glue.webhook.onPost((event) => {
  const body = JSON.parse(event.bodyText || "{}");
  console.log("Webhook payload", body);
});
```

### GitHub event handling

```typescript
import { glue } from "jsr:@streak-glue/runtime";

glue.github.onPullRequestEvent("owner", "repo", (event) => {
  console.log("PR title:", event.payload.pull_request.title);
});
```

### Cron jobs

```typescript
import { glue } from "jsr:@streak-glue/runtime";

glue.cron.everyXMinutes(30, () => {
  console.log("Running scheduled task");
});
```

### Gmail triggers

```typescript
import { glue } from "jsr:@streak-glue/runtime";

glue.gmail.onMessage((event) => {
  console.log("New email subject:", event.subject);
}, {
  accountEmailAddress: "support@example.com",
});
```

### Calling third-party SDKs

When Glue gives you access to a service account, prefer the service's official client library over raw HTTP calls.

```typescript
import { glue } from "jsr:@streak-glue/runtime";
import { WebClient } from "npm:@slack/web-api";

const slackFetcher = glue.slack.createBotMessageSendingCredentialFetcher();

glue.github.onPullRequestEvent("owner", "repo", async (event) => {
  if (event.payload.action !== "opened") {
    return;
  }

  const cred = await slackFetcher.get();
  const slack = new WebClient(cred.accessToken);
  const pr = event.payload.pull_request;

  await slack.chat.postMessage({
    channel: "#engineering",
    text: `New PR opened: <${pr.html_url}|#${pr.number} ${pr.title}> by ${pr.user.login}`,
  });
});
```

## Development Workflow

### Local development

Use `glue dev` for the fastest loop:

```bash
glue dev path/to/your-glue.ts
```

What this gives you:

- Hot reload as you edit the file
- Real events tunneled to your local machine
- A debugger port by default
- Replay support for the last event or a specific execution

### Debugging

Use normal TypeScript debugging techniques:

- `console.log`
- your editor debugger
- Chrome DevTools

If you need to wait for the debugger before running:

```bash
glue dev --inspect-wait path/to/your-glue.ts
```

If you want to disable the debugger:

```bash
glue dev --no-debug path/to/your-glue.ts
```

### Production monitoring

```bash
glue logs -f <glue-name-or-id>
glue describe <glue-name-or-id>
glue describe <execution-id>
```

Use `glue logs` to inspect execution history and live runs, and `glue describe` to inspect Glues, deployments, executions, triggers, and accounts.

## Glue Runtime Surface

The main `glue` object exposes integrations including:

- `glue.webhook`
- `glue.cron`
- `glue.github`
- `glue.gmail`
- `glue.google`
- `glue.drive`
- `glue.sheets`
- `glue.slack`
- `glue.stripe`
- `glue.intercom`
- `glue.streak`
- `glue.resend`
- `glue.openai`
- `glue.debug`

Pick the narrowest integration API that matches the task instead of dropping down to lower-level custom plumbing.

## Packaging and Deployment Notes

Glue deployment bundles your entry file and its local relative imports.

- Keep the Glue entrypoint as a normal local TypeScript file
- Put shared code in local files imported relatively from the entrypoint
- Keep the project layout simple so `glue dev` and `glue deploy` can follow your imports cleanly

## Quick Reference

| Task | Command |
| --- | --- |
| Sign in | `glue login` |
| Check account | `glue whoami` |
| Create a Glue | `glue create` |
| Run locally | `glue dev path/to/file.ts` |
| Replay production event locally | `glue dev --replay <executionId> path/to/file.ts` |
| Deploy | `glue deploy path/to/file.ts` |
| View logs | `glue logs -f <glue>` |
| Inspect resources | `glue describe <id-or-name>` |
| Replay deployed execution | `glue replay <executionId>` |
| List accounts | `glue accounts` |

## Common Mistakes

**Registering handlers dynamically**

```typescript
// ❌ Wrong - do not register handlers after startup in some later code path
async function main() {
  if (Math.random() > 0.5) {
    glue.webhook.onPost(() => {});
  }
}

// ✅ Correct - register handlers at top level during initialization
import { glue } from "jsr:@streak-glue/runtime";

glue.webhook.onPost((event) => {
  console.log(event.bodyText);
});
```

**Skipping the local development loop**

```bash
# ❌ Slower workflow
glue deploy path/to/file.ts

# ✅ Better workflow
glue dev path/to/file.ts
# trigger a real event, iterate, then deploy once behavior looks correct
glue deploy path/to/file.ts
```

**Hardcoding secrets**

```typescript
// ❌ Wrong
const apiKey = "super-secret";

// ✅ Correct
// Use Glue accounts and runtime credential helpers instead of embedding secrets.
```

**Using fake sample data instead of real events when debugging**

```bash
# ❌ Less representative than real event flows
# invent fake payloads first

# ✅ Better
# run glue dev, trigger the real external event, then press `r` to replay it
```

**Ignoring official SDKs**

```typescript
// ❌ Wrong - handwritten auth and API plumbing when an official client exists

// ✅ Correct - prefer the official SDK for GitHub, Slack, Stripe, Google, etc.
// Use Glue runtime integrations and credentials to configure the SDK cleanly.
```

## Documentation

### Glue Documentation

The Glue documentation is available at [docs.glue.wtf](https://docs.glue.wtf/llms-full.txt).

### Glue Runtime API

The Glue runtime API is documented on [JSR](https://jsr.io/@streak-glue/runtime/doc) or by running `deno doc jsr:@streak-glue/runtime`.

### Glue CLI

The Glue CLI is documented on [Mintlify](https://docs.glue.wtf/reference/cli) or by running `glue help` in your terminal.