# Troubleshooting

## A tool call hangs, or the turn ends with it unfinished

The server is not authenticated yet. Before **Sign in with Mirelo** completes, a call does not come
back with a clean error — it simply does not finish.

Do not retry it, and do not go looking for a fault. Ask the user to complete sign-in (see the setup
section in `POWER.md`), then call `get_account` to confirm.

## No Mirelo tools are listed at all

The power installed but the server never connected. Open the MCP servers view in the Kiro panel and
connect `mirelo` from there.

## 403, with a message pointing at https://mirelo.ai/mcp

The account authenticated but does not have MCP access. Every signed-in Mirelo account has it by
default, so this is unusual.

It is deliberately a **403 and not a 401**: the credentials are fine, so re-authenticating will not
help and you should not loop on it. Send the user to <https://mirelo.ai/mcp> or
<support@mirelo.ai>.

## 402 — out of credits

Connecting is free; generating is not. Show what the job would have cost (`preflight` gives the
figure) and point at <https://mirelo.ai/pricing>.

Do not retry. This is expected behaviour rather than a fault, and a second attempt will not succeed.

## The upload is rejected on its signature

The content type sent does not match the one the upload was signed for. Run the command from
`create_upload` exactly as returned; if you rewrote it, that is the cause. Nothing converts a stored
file afterwards, so mint a new upload with the correct type and re-upload.

## The upload fails with a proxy or connection error

The shell has no outbound access — a client setting, not a Mirelo error. Retrying will not help. See
`steering/file-inputs.md` for the two ways round it.

## The upload link stopped working

Upload links are short-lived. Call `create_upload` again and upload straight away.

## "Asset not found" when generating

Either the upload never landed, or the asset belongs to a different Mirelo account. Check with
`inspect_asset` — a not-found error there means the upload did not complete. An asset staged from
another account is rejected on ownership.

## inspect_asset reports no duration

No duration could be read, usually because the file is not decodable media — a truncated upload, or
something that is not audio or video at all. Re-check the file rather than retrying.

## 400 on a retry with an idempotency key

That key was already used with **different** parameters. A key means "the same call again", not
"another call". Use a fresh one for new work.

## A job never finishes, or the wait times out

A wait that times out has **not** cancelled the job. Switch to `get_job` with the same `job_id` and
keep polling. Do not resubmit — that starts a second paid job.

## The result URL will not download

Use the short download link rather than the raw result URL. The raw ones are long enough that some
clients refuse them outright, and they expire on the job's clock rather than on first use.

Re-polling a job returns the same URLs; only re-running it produces new ones.

## The transcript fills with download progress noise

On Windows, `Invoke-WebRequest` emits a progress record per chunk. Set
`$ProgressPreference = 'SilentlyContinue'` before it, or use `curl.exe -sS -o <path> <url>`.

## The user asked for music

Not available on this surface, and the SFX tools are not a substitute. See the music section in
`POWER.md`.

## Getting help

<support@mirelo.ai> · <https://mirelo.ai/support> · product page <https://mirelo.ai/mcp>
