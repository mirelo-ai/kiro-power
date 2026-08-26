# Editing existing audio

Two tools change audio the user already has. Both take the input as a staged asset or as a download
link from an earlier Mirelo job. Their own descriptions carry the parameters and the current limits —
those move with the model version, so read them there rather than trusting a number written here.

## Which tool for which request

Picking wrong still spends credits, so this is the decision worth getting right:

| The user says | Reach for |
| --- | --- |
| "make this cover the whole scene" / "loop it longer" | `extend_audio`, looping if it must be seamless |
| "continue this sound to match the rest of the video" | `extend_audio_with_video` |
| "fix / replace this bit, keep the rest" | `inpaint_audio` |
| "regenerate the whole thing, differently" | `text_to_sfx` or `video_to_sfx` again — not an edit tool |

If the user wants a different take of the whole sound, generate again. Inpainting the entire span is
the expensive way to get a worse version of that.

## Let the server measure the input

The extend tools ask how much **new** audio to add, not what the total should become. Given that, they
probe the existing clip themselves — so you never need `ffprobe`, and you never need to ask the user
how long their file is.

## Inpainting is a span, not a duration

`inpaint_audio` replaces one stretch and leaves everything outside it untouched — the tool for "there
is a glitch from 0:05 to 0:12, fix just that part". It takes two absolute positions rather than a
length. The very start of a file cannot be inpainted, because the model needs context ahead of the
gap; there is no such restriction at the end. A long file is windowed around the gap automatically, so
do not trim the input yourself.

## Chaining without re-uploading

Generate → extend → inpaint needs no re-uploads: pass each finished job's download link as the next
call's input. Those links are signed when they are followed, so a long chain does not race an expiry.

```
text_to_sfx   → link A
extend_audio  (input: link A) → link B
inpaint_audio (input: link B) → final
```
