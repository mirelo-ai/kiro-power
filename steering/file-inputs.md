# Getting a file to Mirelo

Mirelo does not fetch arbitrary URLs. Anything that is not already a Mirelo result must be staged as
an asset first and then referenced by `asset_id`. The tool descriptions cover the rules; this file
covers doing it from Kiro.

## The default path

```
create_upload({
  content_type: "video/mp4",
  filename: "C:/proj/clips/fight.mp4"   # full path; it is shell-quoted, so ~ will not expand
})
→ { asset_id, upload_url, expires_in, max_size_bytes, curl_command }
```

**Run `curl_command` exactly as returned**, then pass `asset_id` to a generate or edit tool.

Kiro executes real local shell commands, so this works with no extra setup — unlike chat clients
whose sandboxes block outbound network by default. It is the reason a large local file is a
non-event here.

Two failure modes worth recognising on sight:

- **`403 SignatureDoesNotMatch`** — the `PUT` sent a different `Content-Type` than the one signed into
  the URL. The generated command already sets the right header; if you rewrote the command, that is
  the cause. Nothing converts a stored file afterwards, so mint a new upload and re-upload.
- **A proxy or connection error** — the shell has no outbound access. A client setting, not a Mirelo
  error, and retrying will not help. Allowlisting `mcp.mirelo.ai` does not help either: tool calls are
  made by the host application and never traverse the shell. Only the upload host matters. The
  fallback is <https://mirelo.ai/studio/mcp-upload>, which needs the same Mirelo account.

`expires_in` is short. Upload immediately; if the URL lapses, call `create_upload` again.

## Picking content_type

Pass what matches the file on disk, not what you wish it were:

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

## Confirming it landed

`inspect_asset({ asset_id })` is **optional** — `video_to_sfx` measures an asset itself and the extend
tools probe the audio prefix. Each call costs an `ffprobe` pass, so it is not a routine pre-flight
step. Call it to confirm the `PUT` landed (a not-found error means it has not) or to see duration and
dimensions before committing credits.

## Reusing a Mirelo result

A prior job's `result.download_urls` entry goes straight back in as an input `url`, with no
re-upload. This is the path to prefer when chaining generate → extend → inpaint. Raw `result_urls`
are download fallbacks only and are not accepted as inputs; if a `download_urls` entry is missing,
download the file and stage it with `create_upload`.

## Inline bytes

`create_asset` exists for files of a few KB. Do not shrink or re-encode audio to fit under its
ceiling — the waveform is what the model conditions on, so degrading it to save one shell command is
a bad trade. Use `create_upload`.
