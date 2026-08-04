# Scoring a video

`video_to_sfx` conditions the generation on the footage itself. Reach for it whenever the user has a
clip — gameplay capture, an animation, a rough cut — and wants sound that lands on what happens on
screen. `text_to_sfx` is for when there is no clip, or the user wants a standalone asset.

The tool's own description carries the parameters, the limits and the rules. Three judgement calls it
cannot make for you:

## Let the server measure the clip

Do not run `ffprobe`, and do not ask the user how long their video is. Hand over the staged asset and
the server measures it, then tells you the window it scored. Reach for a duration only when the user
wants a **shorter** window than the whole clip — that is an editorial choice, and it is what they are
billed for, so it is worth confirming rather than assuming.

If a clip comes back refused for being too long, that is deliberate: trimming it silently would bill
for a window nobody chose. The error names the measurement and the cap, which is what you need in
order to ask the user which part they actually want scored.

## Usually, do not write a prompt

The model already sees the video. A prompt authored by describing the footage is a lossy transcription
of what it can see, and it competes with the real thing — a wrong guess steers the result away from
the picture.

Leave the prompt out when the user asked for nothing specific. Add one only to carry an explicit
instruction: a mood ("sparse and tense"), or a restriction ("only footsteps, no music cues"). Do not
analyse the video in order to author one.

## Do not assume the output is video

Asking for a video output is a request, not a guarantee — a long clip comes back as audio anyway. So
do not infer the artifact kind from what you asked for. Treat a result as audio unless you know that
specific artifact is video.

## Workflow in Kiro

```
1. stage the clip           (steering/file-inputs.md)
2. preflight                quote the cost to the user
3. video_to_sfx             pass the asset; no duration, no prompt
4. get_job                  poll until terminal
5. download into the project's audio folder, report path + cost
```

Step 3 deliberately passes neither a duration nor a prompt: the clip length is the window, and the
video is the instruction.
