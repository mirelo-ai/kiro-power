# Getting a file to Mirelo

Mirelo does not fetch arbitrary URLs. Anything that is not already a Mirelo result has to be staged
as an asset first and then referenced by its `asset_id`. `create_upload` is the tool for that, and its
own description carries the parameters and limits — this file is about doing it from Kiro.

## The shape of it

```
create_upload  →  asset_id + a ready-to-run upload command
run that command in a shell  →  HTTP 200
pass asset_id to a generate or edit tool
```

**Run the command exactly as returned.** It is built for the file path you passed and already sets the
content type the upload was signed for; rewriting it is how it breaks. Two commands come back, one per
shell family — take the one matching the shell you are about to use, which in Kiro's PowerShell
terminal is the PowerShell one.

Kiro executes real local commands, so a large local file is a non-event here. Chat clients that
sandbox their shell cannot do this at all, which is why the browser fallback below exists.

## Choosing the content type

Pass what matches the file on disk, not what you wish it were. A mismatch is rejected at upload time,
and nothing converts a stored file afterwards — you would have to re-upload.

| File | Content type |
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

`inspect_asset` is **optional** — the generate tools measure an asset themselves. Call it to confirm
the upload actually landed, or to see the duration and dimensions before committing credits. It is not
a routine step before every job.

## Reusing a Mirelo result

A finished job's download link can go straight back in as an input, with no re-upload. That is the
path to prefer when chaining generate → extend → inpaint. Only Mirelo's own download links are
accepted as input URLs; anything else has to be downloaded and staged with `create_upload`.

## When the shell has no outbound network

If the upload fails with a proxy or connection error, the shell cannot reach the network. That is a
client setting, not a Mirelo error, and retrying will not help. Allowlisting `mcp.mirelo.ai` does not
help either — tool calls are made by the host application and never traverse the shell. Only the
upload host matters.

The fallback is <https://mirelo.ai/studio/mcp-upload>: the user uploads there and pastes back the
`asset_id`. It needs the same Mirelo account — an asset staged by a different account is rejected.
