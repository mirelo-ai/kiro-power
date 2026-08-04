# Calling Mirelo from the user's own application

Read this only when the goal is **shipping Mirelo inside the user's product** — a backend route, a
game build step, a CI job. For generating audio *for* the project while working in Kiro, stay with
the MCP tools; they need no keys and are already authenticated.

This file is deliberately a pointer, not a copy. The endpoint list, request and response schemas, and
the interactive playground live in the API reference, which is generated from the live specification —
consult it rather than trusting a table transcribed into a power.

**Read the reference first:** <https://mirelo.ai/api-docs> · developer overview:
<https://mirelo.ai/developers>

## What differs from this MCP surface

| | Hosted MCP (this power) | Public HTTP API |
| --- | --- | --- |
| Auth | Sign in with Mirelo (OAuth), per user | `sk-` API key, per project |
| Base | `https://mcp.mirelo.ai/mcp` | `https://api.mirelo.ai` |
| Shape | MCP tools | REST, `/v2/...` |
| Billing | credits on the signed-in account | credits on the key's account |
| Scope | SFX generate and edit | broader — check the reference |

The API is the wider surface. Capabilities absent from this MCP server (music among them) may still
be available over HTTP, so check the reference before telling a user something is impossible.

## Getting a key

API keys are created in Mirelo Studio and start with `sk-`. They are secrets: server-side only, never
in client code, a mobile bundle, or a committed file. Read them from the environment.

**Never put a key in a file you are about to commit.** If the workspace has no ignored env file yet,
create one and add it to `.gitignore` before writing the value.

## Official SDK

```bash
npm install @mirelo/sdk
```

`@mirelo/sdk` is the official Node SDK. Prefer it over hand-rolled `fetch` calls — it tracks the
request and response shapes, so it does not drift the way a copied snippet does.

## Two things the reference will not shout loudly enough

Both are the same mechanics as the MCP tools, and both catch people out:

1. **Generation is asynchronous.** The generate endpoints hand back a job, not audio. Poll the job
   until it is terminal, then read the result URLs. A synchronous variant exists for short work —
   check the reference for which endpoints offer it before assuming.
2. **Retries need an idempotency key.** Without one, a retried create is a second paid job. With the
   same key and the same parameters, you get the original job back.

## Where to send the user

- Reference and playground: <https://mirelo.ai/api-docs>
- Developer overview: <https://mirelo.ai/developers>
- Pricing and credits: <https://mirelo.ai/pricing>
- API terms: <https://mirelo.ai/api-terms>
