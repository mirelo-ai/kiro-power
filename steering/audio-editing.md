# Editing existing audio

Two tools change audio the user already has. Both take input as `{ asset_id }` or as a `{ url }` that
is a Mirelo `download_urls` link from a prior job. The tool descriptions carry the parameter rules;
this file is about choosing the right tool and not measuring things you do not need to measure.

## Which tool for which request

| The user says | Reach for |
| --- | --- |
| "make this cover the whole scene / loop longer" | `extend_audio`, with `loop: true` if it must loop seamlessly |
| "continue this sound to match the rest of the video" | `extend_audio_with_video` |
| "fix / replace this bit, keep the rest" | `inpaint_audio` |
| "regenerate the whole thing, differently" | `text_to_sfx` or `video_to_sfx` again — not an edit tool |

An edit tool called on the wrong intent still spends credits. If the user wants a different take of
the whole sound, generate again rather than inpainting the entire span.

## Do not measure the input audio

Quoted verbatim from the `extend_audio` and `extend_audio_with_video` tool descriptions:

> You never need to measure the input audio: pass append_duration_ms (how much new audio to add) and
> the server probes the prefix with ffprobe and derives the total itself. duration_ms is the legacy
> total (prefix + extension) and is the only reason to know the input length — prefer
> append_duration_ms and do not shell out to ffprobe.

Provide exactly one of the two.

## Limits at a glance

Consolidated because they are split across two tool descriptions. The descriptions are canonical.

| | `extend_audio` | `extend_audio_with_video` |
| --- | --- | --- |
| Input audio prefix | ≥ 3 s | ≥ 3 s |
| `append_duration_ms` | 1000–30000 | 1000–57000 |
| with `loop: true` | ≥ 2000 | — |

A prefix shorter than 3 s cannot condition the model and is rejected. The video-guided variant allows
a longer tail because the picture carries the structure. `loop: true` needs at least 2000 ms of new
audio — less gives the model no room to build a smooth transition back to the start.

## Inpainting

`inpaint_audio` replaces one stretch and leaves everything outside it alone — the tool for "there is a
glitch from 0:05 to 0:12, fix just that part". The region is two absolute positions, `start_ms` and
`end_ms`, not a duration. The first second of a file cannot be inpainted: the model needs prefix
context ahead of the gap. There is no tail restriction, and a long file is windowed around the gap
automatically, so do not trim the input yourself.

## Chaining

Generate → extend → inpaint needs no re-uploads — pass each job's `result.download_urls` entry as the
next call's input `url`:

```
text_to_sfx({ prompt: "rain on a tin roof", duration_ms: 10000, loop: true })  → A
extend_audio({ audio: { url: "<A download_urls[0]>" }, append_duration_ms: 20000 })  → B
inpaint_audio({ audio: { url: "<B download_urls[0]>" }, start_ms: 4000, end_ms: 6500 })
```

Those links are signed when followed, so a long chain does not race an expiry.
