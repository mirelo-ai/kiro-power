# Troubleshooting

## No Mirelo tools appear after installing the power

The OAuth flow did not complete. Reconnect the `mirelo` server and finish **Sign in with Mirelo** in
the browser. Confirm with `get_account` — it returns the signed-in email and credit balance.

## 403, with a message pointing at https://mirelo.ai/mcp

The account authenticated successfully but does not have MCP access. Every signed-in Mirelo account
has it by default, so this is unusual.

Note it is deliberately a **403 and not a 401**: the credentials are fine, so re-authenticating will
not help and the agent should not loop on it. Send the user to <https://mirelo.ai/mcp>, or to
<support@mirelo.ai>.

## 402 — out of credits

Connecting is free; generating is not. The account has no credits left for this job.

Show what the job would have cost (`preflight` gives the figure) and point at
<https://mirelo.ai/pricing>. Do not retry — the charge is not going to succeed on a second attempt,
and this is expected behaviour rather than a fault.

## The upload PUT fails with 403 SignatureDoesNotMatch

The `Content-Type` sent does not match the one signed into the URL.

Run the `curl_command` from `create_upload` exactly as returned — it already sets the right header.
If you rewrote the command, that is the cause. Nothing converts a stored file afterwards, so mint a
new upload with the correct `content_type` and re-upload.

## The upload PUT fails with a proxy or connection error

The shell has no outbound network access. This is a client setting, not a Mirelo error, and retrying
will not help. See `steering/file-inputs.md` for the two ways round it — allowlisting the upload
host and starting a **new** conversation, or the browser upload at
<https://mirelo.ai/studio/mcp-upload>.

Allowlisting `mcp.mirelo.ai` does not help: tool calls are made by the host application and never
traverse the shell.

## The upload URL stopped working

Pre-signed URLs are short-lived (`expires_in` in the `create_upload` response). Call `create_upload`
again and upload straight away.

## "Asset not found" when generating

Either the `PUT` never landed, or the asset belongs to a different Mirelo account.

Check with `inspect_asset({ asset_id })` — a not-found error there means the upload did not complete.
An `asset_id` staged from another account (including one pasted from someone else's browser upload)
is rejected on ownership.

## inspect_asset returns duration_ms: null

No duration could be read. Usually the file is not decodable media — a truncated upload, or something
that is not audio or video at all. Re-check the file rather than retrying. If you know the true
duration, you can supply it yourself where the tool takes one.

## 400 on a retry with an idempotency key

The key was already used with **different** parameters. Keys are scoped to one logical request; reuse
means "the same call again", not "another call". Use a fresh key for new work.

## The job never finishes / wait_job times out

`wait_job` returning a timeout does **not** cancel the job. Switch to `get_job` with the same
`job_id` and keep polling — in Kiro that is the better pattern anyway, since the agent can drive its
own loop without holding a single call open.

## The result URL will not download

Use `result.download_urls`, not `result.result_urls`. Raw result URLs are ~500 characters, which some
clients refuse outright, and they expire an hour after the job ran regardless of when you read them.
Download links are signed when followed and last the session.

Re-polling the job returns the same stored URLs; only re-running the job produces new ones.

## A result cannot be passed back as an input

Only a `download_urls` link is accepted as an input `url`. Raw `result_urls` and third-party URLs are
not. If `download_urls` is absent, download the file and stage it with `create_upload`.

## The user asked for music

Not available on this surface, and `text_to_sfx` is not a substitute. See the music rule in
`POWER.md` — it is quoted there verbatim from the tool descriptions.

## A long video is refused rather than trimmed

Expected. A window longer than the cap would bill for audio nobody asked for, so the request fails
with an error naming the measurement and the cap. Choose a window with `start_offset_ms` and
`duration_ms` — see `steering/video-to-sfx.md`.

## Getting help

<support@mirelo.ai> · <https://mirelo.ai/support> · product page <https://mirelo.ai/mcp>
