# Editing existing audio: extend and inpaint

Two tools change audio the user already has. Both take the input as `{ asset_id }` or as a
`{ url }` that is a Mirelo `download_urls` link from a prior job.

## Extending a clip

`extend_audio` appends new audio to the end of an existing clip, text-conditioned.
`extend_audio_with_video` does the same but guided by video, so the new tail lands on what happens on
screen.

```
extend_audio({
  audio: { asset_id: "..." },
  append_duration_ms: 12000,     # how much NEW audio to add
  prompt: "keep the same rain, add distant thunder",   # optional
  loop: false,                   # optional
  num_samples: 1,                # optional, 1–4
  idempotency_key: "..."         # optional
})
```

### Use append_duration_ms, not duration_ms

Provide **exactly one** of the two:

- **`append_duration_ms`** — milliseconds of new audio to add. **Prefer this.** The server probes the
  existing prefix itself and derives the total, so you never have to know how long the input is.
- **`duration_ms`** — the legacy **total** (prefix + extension). It is the only field here that
  requires knowing the input's length, and it exists for backwards compatibility.

This is the reason agents reach for `ffprobe` on the extend tools. They do not need to: pass
`append_duration_ms` and the measurement happens server-side, for a `url` input as well as an
`asset_id`.

### Limits

| | `extend_audio` | `extend_audio_with_video` |
| --- | --- | --- |
| Input audio prefix | ≥ 3 s | ≥ 3 s |
| `append_duration_ms` | 1000–30000 | 1000–57000 |
| `append_duration_ms` with `loop: true` | ≥ 2000 | — |
| `start_offset_ms` | — | offset into the video where the prefix begins, default 0 |

A prefix shorter than 3 s cannot condition the model, and the request is rejected. The video-guided
variant allows a longer tail because the picture carries the structure.

`loop: true` needs at least 2000 ms of new audio — a shorter extension gives the model no room to
build a smooth transition back to the start.

## Inpainting a span

`inpaint_audio` replaces one stretch inside a clip and leaves everything outside it untouched. This
is the tool for "there is a glitch from 0:05 to 0:12, fix just that part" — not a regeneration.

```
inpaint_audio({
  audio: { asset_id: "..." },
  start_ms: 5000,
  end_ms: 12000,
  prompt: "clean room tone, no clicks",    # optional
  video: { asset_id: "..." },              # optional, guides the replacement
  num_samples: 1,
  idempotency_key: "..."
})
```

The region is given as two absolute positions, not a duration:

- `start_ms` must be **≥ 1000**. The first second cannot be inpainted — the model needs prefix
  context ahead of the gap.
- The gap width (`end_ms − start_ms`) must be between **1000 and 8000 ms**.
- There is no restriction at the tail: the gap may run to the end of the file.

For a file longer than the model's context, the surrounding audio is windowed automatically around
the gap. You do not need to trim the input yourself.

## Chaining edits

Generate → extend → inpaint works without re-uploading anything. Pass the prior job's
`result.download_urls` entry as the next call's input `url`:

```
text_to_sfx({ prompt: "rain on a tin roof", duration_ms: 10000, loop: true })
  → job A → download_urls[0]

extend_audio({ audio: { url: "<download_urls[0]>" }, append_duration_ms: 20000 })
  → job B → download_urls[0]

inpaint_audio({ audio: { url: "<B download_urls[0]>" }, start_ms: 4000, end_ms: 6500 })
```

Those links are signed at the moment they are followed, so a long chain does not race an expiry.
Raw `result_urls` are not accepted as inputs — if a `download_urls` entry is missing, download the
file and stage it with `create_upload`.

## Which tool for which request

| The user says | Reach for |
| --- | --- |
| "make this loop longer / cover the whole scene" | `extend_audio` with `loop: true` if it must loop seamlessly |
| "continue this sound to match the rest of the video" | `extend_audio_with_video` |
| "fix / replace this bit, keep the rest" | `inpaint_audio` |
| "regenerate the whole thing, differently" | `text_to_sfx` or `video_to_sfx` again — not an edit tool |

An edit tool called on the wrong intent still spends credits. If the user wants a different take of
the whole sound, generate again rather than inpainting the entire span.
