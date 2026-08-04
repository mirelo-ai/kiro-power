# Scoring a video with video_to_sfx

`video_to_sfx` conditions the generation on the footage itself. It is the tool to reach for whenever
the user has a clip — gameplay capture, an animation, a rough cut — and wants sound that lands on
what happens on screen.

```
video_to_sfx({
  video: { asset_id: "..." },     # exactly one of asset_id or url
  start_offset_ms: 0,             # optional, defaults to 0
  duration_ms: undefined,         # optional with asset_id — see below
  output: "audio",                # optional: "audio" | "video"
  prompt: undefined,              # usually omit — see below
  num_samples: 1,                 # optional, 1–4, multiplies cost
  idempotency_key: "..."          # optional, for safe retries
})
→ { job_id: "...", resolved_duration_ms: 8400 }
```

## duration_ms is a window you choose, not a fact about the file

`duration_ms` and `start_offset_ms` together select **which stretch of video to score**. It is an
editorial decision, and it is what you are billed for — so it is never guessed.

**With `asset_id`, omit it.** The server measures the uploaded file with one real `ffprobe` pass and
scores the rest of the clip from `start_offset_ms`. The window actually used comes back as
`resolved_duration_ms`. This is exact for every format accepted, MP3 containers included.

Pass `duration_ms` explicitly only when you want a **shorter** window than the remainder — for
example, the user asked for sound on just the first three seconds.

**With `url`** (a chained `download_urls` link), `duration_ms` is **required**. A remote length has to
be read out of the container header, which some formats do not carry, and the read is skipped
entirely if the origin is slow — so there would be nothing reliable to bill against.

Accepted range is 1000–600000 ms.

### Never shell out to ffprobe for it

An agent that runs `ffprobe` to fill in `duration_ms` is doing work the server already does, on a
number it does not need to supply. Omit the field with an `asset_id`. If you want the measurement
visible before credits are committed, call `inspect_asset` — it reports the same figure.

### A long clip is refused, not trimmed

If the remainder from `start_offset_ms` is longer than the 600000 ms cap, the request is **refused**
with an error naming both the measurement and the cap. It is not silently clamped, because trimming
would bill the user for a window nobody chose.

A 40-minute upload with `duration_ms` omitted is a request for a 40-minute generation. When you get
that error, you have the numbers you were missing: pick a window with `start_offset_ms` +
`duration_ms`, or ask the user which part of the clip they actually want scored.

## Usually, do not write a prompt

The model already conditions on the video. A `prompt` an agent authors by *describing the footage* is
a lossy transcription of what the model can see, and it competes with the real thing. A wrong guess
actively steers the result away from the picture.

**Omit `prompt` when the user asked for nothing specific.** Pass it only to carry an explicit user
instruction — a mood ("keep it sparse and tense"), or a restriction ("only footsteps, no music
cues"). Do not analyse the video in order to author one.

Max 5000 characters, though short is better.

## output: audio vs video

`output: "video"` asks for a muxed file with the audio laid against the picture. It is a request, not
a guarantee — a long clip comes back as audio regardless.

So do not infer the artifact kind from the flag you sent. Treat a `download_urls` entry as **audio**
unless you specifically know that artifact is video, and only pass a result onward as a video `url`
when the result itself is video.

## Worked example

A user drops a gameplay clip in the workspace and asks for impact sounds.

```
1. create_upload({ content_type: "video/mp4", filename: "<abs path>/gameplay.mp4" })
2. run the returned curl_command → HTTP 200
3. preflight({ endpoint_key: "video-to-sfx/v1.6", duration_ms: 10000 })    # optional, to quote cost
4. video_to_sfx({ video: { asset_id: "..." } })                            # window = whole clip
5. get_job({ job_id }) until status === "succeeded"
6. download result.download_urls[0] into the project's audio folder
7. report the path and the credits spent
```

Step 4 deliberately passes no `duration_ms` and no `prompt`: the clip length is the window, and the
video is the instruction.
