# E2B

## E2B SDK

AgentENV exposes an E2B-compatible API, so the official [E2B SDK](https://github.com/e2b-dev/e2b) works out of the box.

### General Settings

Set environment variables to point at your AgentENV server. See [Environment Variables](../configuration/env-vars.md) for values per deployment mode.

```bash
# Single-node example
export E2B_API_URL=http://127.0.0.1:8000
export E2B_SANDBOX_URL=${E2B_API_URL}
export E2B_API_KEY=${AENV_API_KEY}
```

AgentENV returns `trafficAccessToken` when `network.allowPublicTraffic` is false
and (for secure sandboxes)
`envdAccessToken` for envd control traffic. These credentials have different
headers and trust boundaries: use `e2b-traffic-access-token` for private
application routes and `X-Access-Token` only for envd. Public application
routes require neither token.

### TypeScript SDK

#### Setup

Install the SDK:

```bash
npm install e2b
```

#### Usage

```typescript
import { Sandbox } from "e2b";

// Create a sandbox from a template
const sandbox = await Sandbox.create("<template-id>", {
  apiKey: process.env.E2B_API_KEY,
});

// List running sandboxes
const running = Sandbox.list({
  apiKey: process.env.E2B_API_KEY,
  limit: 20,
  query: { state: ["running"] },
});
console.log(await running.nextItems());

// Run a command inside the sandbox
sandbox.commands.run("echo hello world");

// Pause the sandbox
await Sandbox.Pause(sandbox.sandboxId, {
  apiKey: process.env.E2B_API_KEY,
});

// Kill the sandbox
await sandbox.kill();
```

Replace `<template-id>` with a template that exists in your local template store. Use `e2b template list` or `GET /v2/templates` to see available templates.

### Volume mounts

The TypeScript SDK can create a volume and pass it directly when creating a
sandbox:

```typescript
import { Sandbox, Volume } from "e2b";

const volume = await Volume.create("workspace-volume", {
  apiKey: process.env.E2B_API_KEY,
});
const sandbox = await Sandbox.create("<template-id>", {
  apiKey: process.env.E2B_API_KEY,
  volumeMounts: {
    "/workspace": volume,
  },
});
```

For the Python SDK, create the volume with `aenv volume create` or
`POST /volumes`, then pass its name when creating a sandbox:

```python
sandbox = Sandbox.create(
    "<template-id>",
    volume_mounts={"/workspace": "workspace-volume"},
)
```

AgentENV supports the TypeScript SDK's volume create, list, and delete operations, and accessing the mounted filesystem through the sandbox.
The E2B SDK's direct volume content API is not supported.

### Python SDK

#### Setup

Install the SDK:

```bash
pip install e2b
```

#### Usage

```python
from e2b import Sandbox, SandboxQuery, SandboxState

# Reuse the environment variables set in your shell:
# E2B_API_URL / E2B_SANDBOX_URL / E2B_API_KEY

# Create a sandbox from a template
sandbox = Sandbox.create("<template-id>")

# List running sandboxes
running = Sandbox.list(
    limit=20,
    query=SandboxQuery(state=[SandboxState.RUNNING]),
)
print(running.next_items())

# Run a command inside the sandbox
result = sandbox.commands.run("echo hello world")
print(result.stdout, end="")

# Pause the sandbox
sandbox.beta_pause()

# Kill the sandbox
sandbox.kill()
```

## Process output logs

envd inside every sandbox can ship each process's stdout and stderr, together
with its own log lines, to an HTTP collector. Set `[envd].logs_collector_address`
(or `AENV_ENVD_LOGS_COLLECTOR_ADDRESS`) on the runtime nodes; AgentENV hands
the URL to the guest through the MMDS `address` field on every start, resume,
and fork, so existing templates pick it up without a rebuild. envd POSTs one
JSON object per request, batched per process every 2 seconds or 64 KiB, for
example:

```json
{"level":"info","logger":"process","event_type":"stderr",
 "data":"Traceback (most recent call last):\n","timestamp":"2026-09-25T10:00:00Z",
 "message":"Streaming process event","instanceID":"<sandbox-id>","envID":"<template-id>"}
```

The request carries no credentials; put the collector behind a URL only the
sandbox network can reach, or embed a token in the URL. A private collector
address also has to be allowed by `[network.egress]`. An HTTPS collector needs
a trust store envd can use: tools drives built from this repository ship a CA
bundle and run envd with `SSL_CERT_FILE` set to it, so the guest image does
not need `ca-certificates`; older tools drives rely on the guest's
`/etc/ssl/certs`, and a bare image logs `x509: certificate signed by unknown
authority` in `/var/log/agentenv/envd.stderr.log` instead of shipping.

## E2B CLI

AgentENV is compatible with the E2B CLI, but we recommend using the
[aenv CLI](../getting-started/aenv-cli/index.md) for AgentENV workflows.
