# Sending files to Mirelo

Mirelo does not fetch arbitrary URLs. Anything that is not already a Mirelo result has to be staged
as an **asset** first, and then referenced by `asset_id`.

## The default path: create_upload

```
create_upload({
  content_type: "video/mp4",
  filename: "C:/proj/clips/fight.mp4"   # full path; it is shell-quoted, so ~ will not expand
})
→ {
    asset_id: "...",
    upload_url: "https://...",          # pre-signed, single purpose
    expires_in: 300,                    # seconds; read it from the response, do not assume
    max_size_bytes: 52428800,           # likewise — this path takes files of real size
    curl_command: "curl -sS -X PUT -T '...' -H 'Content-Type: video/mp4' -w 'HTTP %{http_code}\n' '...'"
  }
```

Run `curl_command` **as given**. Then pass `asset_id` to a generate or edit tool.

Four things about this path that are easy to get wrong:

1. **`content_type` is signed into the URL.** A `PUT` that sends a different type, or none, is
   rejected by storage with `403 SignatureDoesNotMatch` at upload time. The generated
   `curl_command` already sets the right header — do not rewrite it.
2. **Nothing converts a file after the fact.** A wrong stored content type is never repaired. Call
   `create_upload` again with the correct type and re-upload.
3. **The URL expires** after `expires_in` seconds. Upload immediately; if it lapses, mint a new one.
4. **Success is an empty 200 body.** That is why `curl_command` prints the status line. `HTTP 200`
   means it landed.

Kiro runs commands on the local machine, so this path normally works with no extra setup — unlike
chat clients whose sandboxes block outbound network by default.

## Confirming the upload

`inspect_asset({ asset_id })` returns `duration_ms`, `content_type`, `size_bytes`, and `width` /
`height` for video.

It is **optional**. `video_to_sfx` measures an `asset_id` itself and the extend tools probe the audio
prefix, so do not call it as a routine step before every job — each call spends an `ffprobe` pass.
Call it when you want to:

- confirm the `PUT` actually landed (a not-found error means it has not), or
- see the real duration or dimensions before committing credits.

A `null` `duration_ms` means no duration could be read — usually the file is not decodable media.
Re-check the file rather than retrying blindly.

## Chaining: reusing a Mirelo result

A prior job's `result.download_urls` entry can be passed straight back as an input `url`, with no
re-upload:

```
extend_audio({ audio: { url: "https://mirelo.ai/dl/..." }, append_duration_ms: 8000 })
```

This is the path to prefer in a generate → extend → inpaint chain: those links are signed when they
are followed, so a long chain does not have to outrun an expiry clock.

Raw `result_urls` are **not** accepted as inputs — they are download fallbacks only. If a
`download_urls` entry is missing, download the file and stage it with `create_upload`.

## Inline bytes: create_asset

`create_asset({ content_type, bytes_base64 })` exists for files of a few KB — the ceiling is about
96 KB decoded, because the bytes have to be emitted into the conversation and then cross the
server's request body.

**Inline is a trap for audio.** Uncompressed WAV at the shortest length the edit tools accept is
already well past the limit. A short, heavily compressed clip does fit — but do not shrink or
re-encode audio to squeeze under the ceiling. The waveform is what the model conditions on, so
degrading it to save one shell command is a bad trade. Use `create_upload`.

## When the shell has no outbound network

If the `PUT` fails with a proxy or connection error, the shell has no outbound access. That is a
client setting, not a Mirelo error, and retrying will not help. Two ways out:

1. Ask the user to allow outbound requests to the upload host, then start a **new** conversation —
   the setting does not apply retroactively to the current one.
2. Ask them to upload at <https://mirelo.ai/studio/mcp-upload> and paste back the `asset_id` it
   shows. That page requires the same Mirelo account, and an `asset_id` staged by a different account
   is rejected on ownership.

Note that allowlisting `mcp.mirelo.ai` does **not** fix this: tool calls are made by the host
application and never traverse the shell. Only the upload host matters.

## Choosing content_type

Pass the type that matches the file on disk, not the type you wish it were:

| File | `content_type` |
| --- | --- |
| `.mp4` | `video/mp4` |
| `.mov` | `video/quicktime` |
| `.webm` | `video/webm` |
| `.wav` | `audio/wav` |
| `.mp3` | `audio/mpeg` |
| `.m4a` / `.aac` | `audio/mp4` / `audio/aac` |
| `.flac` | `audio/flac` |
| `.ogg` | `audio/ogg` |
