---
name: mirelo-sfx
description: Generate and edit sound effects, Foley and ambience with the hosted Mirelo MCP server — score a video clip from the footage itself, create SFX from a text prompt, or extend and inpaint audio you already have. Use when the task involves sound effects, game audio, ambience beds, adding or syncing sound to video or gameplay footage, lengthening an audio clip, or repairing a damaged section of audio.
license: MIT
metadata:
  author: Mirelo
  version: "1.0.0"
---

# Mirelo SFX

## Overview

Mirelo generates sound for video. Give it a clip and it scores what is actually happening on
screen — footsteps, impacts, cloth, room tone — or describe a sound in text and get it back as a
file. It also edits audio you already have: extend a clip to cover a longer scene, or regenerate one
bad stretch in place while the rest stays untouched.

The tools arrive from the hosted Mirelo MCP server. There is no API key to create, store or rotate:
you sign in through the browser once, and generations bill the credits on that Mirelo account — the
same balance the web app uses.

**Reach for this skill when the task involves** adding sound effects or Foley to video or gameplay
footage, building an ambience bed for a scene, generating one-off SFX assets for a game or app,
lengthening an existing audio clip, or repairing a damaged section of audio.

**Key capabilities**

- **Video to SFX** — sound conditioned on the footage itself, synced to what happens on screen
- **Text to SFX** — sound effects, Foley and ambience from a written prompt, optionally seamless-looping
- **Extend** — add new audio to the end of an existing clip, with or without video guidance
- **Inpaint** — replace a chosen span of audio, leaving everything outside it alone
- **Cost before you spend** — credit and latency estimates up front, from the same table that bills

## Getting set up

### 1. You need a Mirelo account

Free to create at <https://mirelo.ai>. Signing up and connecting cost nothing; only generation spends
credits, and a new account starts with some. If the user has no account, send them there first — the
sign-in step below cannot create one for them.

### 2. Sign in with Mirelo

Installing this power registers the `mirelo` MCP server. It authenticates with OAuth in the browser,
so the first tool call cannot succeed until that has been completed:

- Kiro opens a browser window for **Sign in with Mirelo**. Complete it and return to the IDE.
- If no window appeared, open the MCP servers view in the Kiro panel, find `mirelo`, and connect it
  from there. Do not guess at menu paths for the user — say which view to open and let them navigate.

### 3. Confirm it worked

Call `get_account`. It returns the signed-in email and the credit balance, which is the cheapest
proof that both OAuth and MCP access are in place. Show the balance to the user.

**Before sign-in, a tool call does not fail cleanly — it hangs and the turn ends with the call
unfinished.** That is not a bug to retry or diagnose; it means the connection is not authenticated
yet. Ask the user to complete sign-in, then call `get_account` again.

No environment variables, keys or config files are required at any point.

## Available MCP servers

### mirelo

- **Connection:** `https://mcp.mirelo.ai/mcp` (remote, streamable HTTP)
- **Authorization:** OAuth — browser-based **Sign in with Mirelo**, no client credentials to supply
- **Billing:** generations spend credits on the signed-in Mirelo account. Connecting is free.

## Reference files

Load only what the task calls for. Do not read them all up front.

| Working on | Read |
| --- | --- |
| First run, signing in, or a text-prompt sound effect | this file is enough |
| Sending a local video or audio file to Mirelo | `references/file-inputs.md` |
| Scoring a video clip, choosing which part to score | `references/video-to-sfx.md` |
| Extending a clip, or repairing part of one | `references/audio-editing.md` |
| Calling Mirelo from the user's own application code | `references/api-integration.md` |
| An error, or a result that will not download | `references/troubleshooting.md` |

The server sends a full description with every tool, and those descriptions are canonical: they carry
the parameter rules, the current limits and the reasoning, and they are always more current than this
power. Nothing here restates them. This power covers what a tool description cannot — when to reach
for this server, how to get signed in, how to get a file in, where results should land in the project,
and what an error means.

## Tools

Exact names, so you can call them. Parameters come from each tool's own description.

**Read-only**

| Tool | Purpose |
| --- | --- |
| `get_account` | Signed-in email and credit balance |
| `preflight` | Credits and ETA for a planned job, before committing them |
| `inspect_asset` | What an uploaded file actually contains — duration, type, size, and dimensions for video |
| `get_job` | Poll one job |
| `wait_job` | Block until a job finishes |

**Staging a file — no credits**

| Tool | Purpose |
| --- | --- |
| `create_upload` | **Preferred.** Returns an `asset_id` plus a ready-to-run upload command; the bytes go from disk to storage without passing through the conversation |
| `create_asset` | Inline base64, for files of a few KB only |

**Paid, asynchronous**

| Tool | Purpose |
| --- | --- |
| `text_to_sfx` | Sound effects, Foley, ambience from a text prompt |
| `video_to_sfx` | Sound conditioned on a video clip |
| `extend_audio` | Append new audio to an existing clip, text-conditioned |
| `extend_audio_with_video` | Append new audio, guided by video |
| `inpaint_audio` | Replace a span inside an existing clip, optionally video-guided |

## How a job flows

Every paid tool returns a `job_id` immediately. None return audio inline.

1. Call the tool → `{ job_id, ... }`.
2. Poll `get_job` until the status is terminal. **Prefer `get_job` over `wait_job` in Kiro** — the
   agent can drive its own loop, which outlives a single blocking call.
3. On success, save the result into the project.

A `wait_job` that times out has **not** cancelled the job. Go back to `get_job` with the same
`job_id` rather than resubmitting, which would charge again.

## Saving results into the project

This is the reason to generate audio inside an IDE rather than a chat window: the file can land in
the repository, named sensibly, in one turn.

1. Take the first download link off the finished job.
2. Write it into the project's own asset folder — match the convention the workspace already uses
   (`assets/audio/`, `Content/Audio/`, `public/sfx/`, …) rather than inventing one.
3. Name the file after the sound, not the job. `footstep-gravel-01.wav` beats `job-abc123.wav`.
4. Tell the user the path you wrote and what it cost.

**A download link is a redirect, so follow it.** Mirelo's short links `302` to storage. With `curl`
that means **`-L` is mandatory** — without it you get a 0-byte file and exit code 0, which looks like
success until something tries to play it:

```bash
curl -sSL -o <path> <url>          # -L is not optional
```

**On Windows, silence the progress meter.** `Invoke-WebRequest` follows redirects on its own, but
streams a progress record per chunk, which floods the transcript with thousands of
`Writing request stream...` lines and crowds out the rest of the session. Set
`$ProgressPreference = 'SilentlyContinue'` first, or use the `curl.exe -sSL -o` form above.

**Check the size after writing.** A 0-byte or few-hundred-byte file means the redirect was not
followed, not that the generation failed — the job succeeded and re-polling it returns the same link.

## Spending credits responsibly

- **Quote before you spend.** `preflight` reads the same table that bills, so its estimate is not a
  guess. Use it when the cost is not trivial or the user has not explicitly asked for the job, and
  tell them the number.
- **Make retries safe.** The paid tools accept an idempotency key; reuse it on a retry of the same
  request and you get the original job back instead of a second charge.
- **Do not ask for alternatives nobody wanted.** Requesting multiple samples multiplies the cost.

## Music is not on this surface

There is no music tool on this server, and the SFX tools are not a substitute — a musical prompt
spends the user's credits on something they cannot use. If the user asks for music, say it is not
available here rather than approximating it.

Mirelo does generate music, in Mirelo Studio and its editor plugins, so do not report the capability
as missing from the product. Audio-to-MIDI and multi-stem generation are likewise not exposed here.

## Call shapes

Argument **names** only — every rule, bound and default lives in the tool's own description, which is
the current source. This is here so you can form a call without having to guess at field names.

```
get_account({})
preflight({ endpoint_key, duration_ms })          endpoint_key names the operation and model version
inspect_asset({ asset_id })
create_upload({ content_type, filename })         filename is a full local path
create_asset({ content_type, bytes_base64 })      tiny files only

text_to_sfx({ prompt, duration_ms, loop?, num_samples?, idempotency_key? })
video_to_sfx({ video: { asset_id }, start_offset_ms?, duration_ms?, prompt?, output? })
extend_audio({ audio: { asset_id }, append_duration_ms, prompt?, loop? })
extend_audio_with_video({ audio: { asset_id }, video: { asset_id }, append_duration_ms })
inpaint_audio({ audio: { asset_id }, start_ms, end_ms, prompt?, video? })

get_job({ job_id })
wait_job({ job_id })
```

Media arguments take either `{ asset_id }` from `create_upload` or `{ url }` holding a Mirelo download
link from an earlier job. Nothing else is accepted as an input URL.

## Worked example

A user asks for a laser zap and wants the file in the project.

```
preflight({ endpoint_key: "text-to-sfx/v1.6", duration_ms: 1000 })
  → quote the credit cost to the user, then continue
text_to_sfx({ prompt: "laser zap, sharp attack, quick decay", duration_ms: 1000 })
  → { job_id }
get_job({ job_id })
  → poll until terminal, then download into the project's audio folder
  → report the path and the credits spent
```

For a video clip, `references/file-inputs.md` covers staging the file first and
`references/video-to-sfx.md` covers choosing what to score.

## Best practices

**Do**

- Call `get_account` after install to confirm auth and show the balance.
- Use `create_upload` for any real file, and run the command it returns as given.
- Poll with `get_job` rather than holding a blocking call open.
- Say what a job will cost before spending credits the user did not ask for.
- Save results into the workspace and report the path.
- Decline music here, and point at Mirelo Studio instead.

**Don't**

- Don't retry or re-diagnose a tool call that hung before sign-in — get the user signed in.
- Don't resubmit a paid job because a wait timed out; poll it.
- Don't shrink or re-encode a file to squeeze it through the inline path — the waveform is what the
  model conditions on.
- Don't invent Kiro menu paths when guiding the user; name the view and let them navigate.

## Requirements

- A Mirelo account (free at <https://mirelo.ai>). Credits are spent by generation, not by connecting.
- Outbound network access from the shell for the upload. Kiro runs real local commands, so this
  normally works out of the box; `references/file-inputs.md` has the fallbacks if it does not.
- MCP access on the account. Every signed-in Mirelo account has it by default; a `403` pointing at
  <https://mirelo.ai/mcp> means this one does not.

## Resources

- Product page and connect guides: <https://mirelo.ai/mcp>
- Developer docs and API reference: <https://mirelo.ai/api-docs>
- Pricing and credits: <https://mirelo.ai/pricing>
- Browser upload, when a sandbox blocks the upload: <https://mirelo.ai/studio/mcp-upload>

## Licensing, privacy and support

- **This power** (the documentation in this repository) is released under the MIT License. See
  `LICENSE`.
- **The Mirelo MCP server and the Mirelo service** are governed by the Mirelo Terms of Service
  (<https://mirelo.ai/terms>), the API Terms (<https://mirelo.ai/api-terms>) and the Acceptable Use
  Policy (<https://mirelo.ai/acceptable-use-policy>).
- **Privacy:** <https://mirelo.ai/privacy>. Media you upload is processed to fulfil the generation
  request against the account you signed in with.
- **Security:** <https://mirelo.ai/security>.
- **Support:** <support@mirelo.ai>, or <https://mirelo.ai/support>. Issues with this power's
  documentation can also be filed in this repository.
