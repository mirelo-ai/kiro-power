# Scoring a video

`video_to_sfx` conditions the generation on the footage itself. Reach for it whenever the user has a
clip — gameplay capture, an animation, a rough cut — and wants sound that lands on what happens on
screen. `text_to_sfx` is for when there is no clip, or the user wants a standalone asset.

The tool description carries the parameter rules and the current limits. Two things below are worth
having in front of you anyway.

## Do not measure the clip yourself

Quoted verbatim from the `video_to_sfx` tool description:

> Do not shell out to ffprobe for duration_ms. With asset_id you can omit it: we measure the uploaded
> file with one ffprobe pass — exact for every format we accept — and score the rest of the clip from
> start_offset_ms. Pass duration_ms whenever you want a shorter window than that, since the window is
> what you are billed for. With url (a chained download_urls link) you must pass duration_ms, because
> a remote length comes from the container header, which some formats do not carry. inspect_asset
> reports the same measurement up front if you want it before committing credits, but you no longer
> have to call it first.

The window actually used comes back as `resolved_duration_ms`. A remainder longer than the cap is
**refused rather than trimmed**, because trimming would bill for a window nobody chose — the error
names both the measurement and the cap, so when you hit it you have the numbers you were missing.
Pick a window with `start_offset_ms` + `duration_ms`, or ask the user which part they want scored.

## Do not write a prompt from the footage

The model already sees the video. A `prompt` authored by describing the footage competes with the
real thing, and a wrong guess steers the result away from the picture.

Omit `prompt` when the user asked for nothing specific. Pass it only to carry an explicit
instruction — a mood ("sparse and tense"), or a restriction ("only footsteps"). Do not analyse the
video in order to author one.

## Workflow in Kiro

```
1. create_upload({ content_type: "video/mp4", filename: "<abs path>/gameplay.mp4" })
2. run the returned curl_command → HTTP 200
3. preflight({ endpoint_key: "video-to-sfx/v1.6", duration_ms: 10000 })   # optional, to quote cost
4. video_to_sfx({ video: { asset_id } })                                  # window = rest of clip
5. get_job({ job_id }) until status === "succeeded"
6. download result.download_urls[0] into the project's audio folder
7. report the path and the credits spent
```

Step 4 passes no `duration_ms` and no `prompt` on purpose: the clip length is the window, and the
video is the instruction.

Asking for `output: "video"` is a request, not a guarantee — a long clip comes back as audio anyway.
So do not infer the artifact kind from the flag you sent.
