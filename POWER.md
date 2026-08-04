---
name: "mirelo"
displayName: "Mirelo Video Sound"
description: "Generate and edit sound effects, Foley and ambience for video, straight from the agent. Score a clip from the video itself, or extend and inpaint audio you already have. Sign in with Mirelo once — no API key to manage."
keywords: ["sound effects", "sfx", "foley", "ambience", "game audio", "video to sfx", "audio generation", "sound design", "mirelo"]
author: "Mirelo"
---

# Mirelo Video Sound

## Overview

Mirelo generates sound for video. Give it a clip and it scores what is actually happening on
screen — footsteps, impacts, cloth, room tone — or describe a sound in text and get it back as a
file. It also edits audio you already have: extend a clip to cover a longer scene, or regenerate one
bad stretch in place while the rest stays untouched.

This power connects the hosted Mirelo MCP server. Install it, complete **Sign in with Mirelo** in
the browser, and the tools are live. There is no API key to create, store, or rotate — generations
bill to the credits on the Mirelo account you signed in with, the same balance the web app uses.

**Use this power when the task involves:** adding sound effects or Foley to video or gameplay
footage, building ambience beds for a scene, generating one-off SFX assets for a game or app,
lengthening an existing audio clip, or repairing a damaged section of audio.

**Key capabilities**

- **Video to SFX** — sound conditioned on the footage itself, synced to what happens on screen
- **Text to SFX** — sound effects, Foley and ambience from a written prompt, optionally seamless-looping
- **Extend** — add new audio to the end of an existing clip, with or without video guidance
- **Inpaint** — replace a chosen span of audio, leaving everything outside it alone
- **Cost before you spend** — credit and latency estimates up front, from the same table that bills

## Onboarding

1. Install this power. Kiro registers the `mirelo` MCP server for you.
2. Complete **Sign in with Mirelo** in the browser window Kiro opens. You need a Mirelo account —
   sign up free at <https://mirelo.ai>.
3. Confirm the connection by calling `get_account`. It returns the signed-in email and the credit
   balance. If tools are missing, the OAuth flow did not finish; reconnect the server.

No environment variables, keys, or config files are required.

## Available MCP servers

### mirelo

- **Connection:** `https://mcp.mirelo.ai/mcp` (remote, streamable HTTP)
- **Authorization:** OAuth — browser-based **Sign in with Mirelo**, no client credentials to supply
- **Billing:** generations spend credits on the signed-in Mirelo account. Connecting is free.

## Steering files

Load only the file that matches the task. Do not read them all up front.

| Working on | Read |
| --- | --- |
| Sending a local video or audio file to Mirelo | `steering/file-inputs.md` |
| Scoring a video clip, choosing the window to score | `steering/video-to-sfx.md` |
| Extending a clip, or repairing part of one | `steering/audio-editing.md` |
| Calling Mirelo from the user's own application code | `steering/api-integration.md` |
| An error, or a result that will not download | `steering/troubleshooting.md` |

## Tools

### Read-only

| Tool | Purpose |
| --- | --- |
| `get_account` | Signed-in email and credit balance |
| `preflight` | Credits and ETA for a planned job, before committing them |
| `inspect_asset` | What an uploaded file actually contains: `duration_ms`, `content_type`, `size_bytes`, plus `width` / `height` for video |
| `get_job` | Poll one job by `job_id` |
| `wait_job` | Block until a job finishes |

### Staging a file (no credits)

| Tool | Purpose |
| --- | --- |
| `create_upload` | **Preferred.** Returns an `asset_id`, a pre-signed `upload_url` and a ready-to-run `curl_command`. Bytes go from disk to storage without passing through the conversation |
| `create_asset` | Inline base64, for files of a few KB only (~96 KB decoded ceiling) |

### Paid, asynchronous

| Tool | Purpose |
| --- | --- |
| `text_to_sfx` | Sound effects, Foley, ambience from a text prompt |
| `video_to_sfx` | Sound conditioned on a video clip |
| `extend_audio` | Append new audio to an existing clip, text-conditioned |
| `extend_audio_with_video` | Append new audio, guided by video |
| `inpaint_audio` | Replace a span inside an existing clip, optionally video-guided |

Every paid tool returns a `job_id` immediately. None of them return audio inline.

## The job model

1. Call a paid tool → it returns `{ job_id, ... }`.
2. Poll with `get_job` until `status` is `succeeded` or `errored`. **Prefer `get_job` in Kiro** — the
   agent can drive its own loop, which survives longer than a single blocking call.
3. On success, read `result.download_urls` and save the file into the workspace.

`wait_job` blocks server-side instead, for hosts that cannot loop. Its `poll_interval_ms` floor is
500 ms (default 1000, max 10000) and `timeout_ms` defaults to 240000 with a 300000 ceiling. A job
that outlives the wait is **not** cancelled — go back to `get_job` with the same `job_id`.

### Reading a result

Prefer **`result.download_urls`** over `result.result_urls`:

- They are short links on `mirelo.ai`, index-aligned with `result_urls`.
- Each is signed at the moment it is followed, so it keeps working for the whole session. Raw
  `result_urls` are ~500 characters — long enough that some clients refuse them outright — and they
  expire an hour after the job ran, regardless of when you read them.
- A `download_urls` entry can be passed straight back as an input `url` to `extend_audio` or
  `inpaint_audio`. A raw `result_urls` value cannot.

`download_urls` is best-effort and may be absent. When it is, download via `result_urls` and, if you
need the file as an input again, stage it with `create_upload`.

## Credits

Call `preflight` before a paid job when the cost matters — it reads the same table that bills, so the
estimate is not a guess. `endpoint_key` is one of `text-to-sfx/v1.6`, `video-to-sfx/v1.6`,
`extend-audio/v1.6`, `extend-audio/with_video/v1.6`, `inpaint-audio/v1.6`,
`inpaint-audio/with_video/v1.6`. Pass `loop: true` for a looped text or extend job so the ETA uses
the loop profile.

**Retries.** Every paid tool takes an optional `idempotency_key` (1–128 chars, `A–Z a–z 0–9 . _ : ~ + / -`).
Reuse the same key on a retry of the same logical request and you get the original `job_id` back
instead of a second charge. Reusing a key with *different* parameters is a `400` — use a new key for
new work.

`num_samples` (1–4) multiplies both the result count and the cost. Leave it unset unless the user
asked for alternatives.

## Music is not available here

There is no music tool on this server, and `text_to_sfx` is not a substitute. A musical prompt sent
to an SFX model spends the user's credits on something they cannot use.

If the user asks for music, **say it is not available through this MCP server** rather than
approximating it. Mirelo does generate music — in Mirelo Studio and its editor plugins — so do not
report the capability as missing from the product.

Audio-to-MIDI and multi-stem generation are likewise not exposed here.

## Saving results into the project

The point of running this in an IDE is that generated audio can land directly in the repository.
After a job succeeds:

1. Take the first `result.download_urls` entry.
2. Download it into the project's own asset folder — match whatever convention the workspace already
   uses (`assets/audio/`, `Content/Audio/`, `public/sfx/`, …) rather than inventing one.
3. Name the file after the sound, not the job. `footstep-gravel-01.wav` beats `job-abc123.wav`.
4. Tell the user the path you wrote and what it cost.

Treat a `download_urls` entry as **audio** unless you specifically know that artifact is a muxed
video file. Asking for `output: "video"` does not guarantee one — a long clip comes back as audio
anyway — so only pass a result onward as a video `url` when the result itself is video.

## Tool usage examples

### Text to SFX

```
text_to_sfx({
  prompt: "glass bottle shattering on a tile floor, close mic, dry room",
  duration_ms: 3000
})
→ { job_id: "..." }

get_job({ job_id: "..." })
→ { status: "succeeded", result: { download_urls: ["https://mirelo.ai/dl/..."], ... } }
```

`duration_ms` is 1000–60000. With `loop: true` it is 3000–600000 and the result is
seamless-looping — the right choice for an ambience bed that has to cover a long scene.
`prompt` accepts up to 5000 characters, but short concrete phrases work best; long prompts are
shortened before they reach the model.

### Video to SFX

```
create_upload({ content_type: "video/mp4", filename: "C:/proj/clips/fight.mp4" })
→ { asset_id: "...", upload_url: "...", curl_command: "curl -sS -X PUT -T ... " }

# run curl_command in a shell, then:
video_to_sfx({ video: { asset_id: "..." } })
→ { job_id: "...", resolved_duration_ms: 8400 }
```

Omit `duration_ms` when passing an `asset_id` — the server measures the clip and scores the rest of
it from `start_offset_ms`, reporting the window it used as `resolved_duration_ms`. See
`steering/video-to-sfx.md` before choosing a window by hand.

### Extend an existing clip

```
extend_audio({
  audio: { asset_id: "..." },
  append_duration_ms: 12000
})
```

Pass `append_duration_ms` (how much new audio to add), not `duration_ms`. See
`steering/audio-editing.md`.

## Best practices

**Do**

- Call `get_account` once after install to confirm auth and show the user their balance.
- Use `create_upload` for any real file, and run the `curl_command` it hands you verbatim.
- Prefer `get_job` polling over `wait_job`.
- Prefer `download_urls` over `result_urls`, and pass `download_urls` entries back when chaining.
- Pass an `idempotency_key` on anything you might retry.
- Say what a job will cost before spending credits the user did not ask you to spend.
- Save results into the workspace and report the path.

**Don't**

- Don't shell out to `ffprobe` to measure a duration. `video_to_sfx` measures an `asset_id` itself,
  and the extend tools probe the audio prefix. See `steering/video-to-sfx.md`.
- Don't write a `prompt` for `video_to_sfx` by describing the footage — the model already sees it,
  and your description competes with what it can see. Pass `prompt` only to carry an explicit user
  instruction.
- Don't approximate music with `text_to_sfx`.
- Don't pass a raw `result_urls` value as an input `url`; it is rejected.
- Don't re-encode or shrink a file to fit `create_asset`. The waveform is what the model conditions
  on — use `create_upload` instead.
- Don't set `num_samples` above 1 unless the user wants alternatives; it multiplies the cost.

## Requirements

- A Mirelo account (free to create at <https://mirelo.ai>). Credits are consumed by generation, not
  by connecting.
- Outbound network access from the shell for the pre-signed upload `PUT`. Kiro runs local commands,
  so this normally works out of the box; see `steering/file-inputs.md` for the fallbacks if it does
  not.
- MCP access on the account. Every signed-in Mirelo account has it by default; a `403` pointing at
  <https://mirelo.ai/mcp> means this account does not.

## Resources

- Product page and connect guides: <https://mirelo.ai/mcp>
- Developer docs and API reference: <https://mirelo.ai/api-docs>
- Pricing and credits: <https://mirelo.ai/pricing>
- Browser upload (when a sandbox blocks the `PUT`): <https://mirelo.ai/studio/mcp-upload>

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
