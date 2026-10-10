---
name: segmind-api
description: Write application code that calls the Segmind API (the segmind Python SDK, or plain HTTP from TypeScript, JavaScript, curl or any language), covering model calls, async submit-and-poll, file uploads, Pixelflows and error handling. Use when the user wants their own app, backend or script to use Segmind. To generate media right now in this session, use the segmind skill instead.
metadata:
  author: Segmind
  short-description: Write code that calls the Segmind API
---

# Segmind API

## Before writing code

- **Get the model's real contract.** Call the Segmind MCP tool `get_model_spec(slug)`, or
  read `https://www.segmind.com/models/<slug>/llms.txt`. Use its endpoint, parameter
  names, enum values and output format exactly. Pick or validate slugs with
  `list_models`; never hard-code a slug or price from memory.
- **The API key belongs to the app, not to you.** The app reads `SEGMIND_API_KEY` from its
  environment at run time. Users create keys at https://platform.segmind.com/api-keys.
  - Never hard-code a key, commit one, or print one.
  - Never call Segmind from browser or mobile client code, where the key would ship to
    users. Call it from a server route, a worker or a script.
  - If you add a `.env.example`, use a placeholder value, and make sure `.env` is in
    `.gitignore`.
  - Do not open or read the user's existing `.env` or secret files yourself.
- **Running the code spends credits.** Ask before executing it against the live API, and
  smoke-test with the cheapest suitable model (`list_models` with `sort="cost"`).

## Choose the call pattern

| Pattern | Endpoint | Use for |
| --- | --- | --- |
| Sync (v1) | `POST https://api.segmind.com/v1/<slug>` | Fast image and audio models inside a request handler. Media models return the file bytes, with the media type in `Content-Type`; text models return JSON. |
| Async (v2) | `POST https://api.segmind.com/v2/<slug>`, then poll | Video, 3D, long audio, anything that can take more than ~30 s, and batches. Never hold a user's HTTP request open for a video job. Submit, store `request_id`, and poll from a worker or the client. |

Every request sends the header `x-api-key: <key>` and a JSON body of the model's parameters.

**v2 lifecycle.**
1. Submit returns `{request_id, status: "QUEUED", status_url, response_url}`.
2. Poll `GET status_url` until `status` is `COMPLETED` or `FAILED`.
3. A FAILED request is served as HTTP 422, with `status: "FAILED"` and an `error` string in
   the body. Treat that body as the answer, not as a transport error.
4. On COMPLETED, `GET response_url` returns the result. It always has `status`, `metrics`
   (including `cost`) and `output`. The rest varies by model; many image models put files
   under `images[].url`. Read the spec or log one response rather than assuming keys.

**Errors.**

| Status | Meaning | What to do |
| --- | --- | --- |
| 400 | Bad parameters | Fix the input |
| 401 | Missing or invalid key | Check the key |
| 404 | Unknown model slug | Check the slug |
| 406 | Insufficient credits | Top up the account; do not retry |
| 429 | Rate limited | Back off and retry |
| 5xx | Transient server error | Retry with backoff |

## Python: use the SDK

`pip install segmind` (1.3+). It reads `SEGMIND_API_KEY` from the environment.

```python
import segmind
from segmind import InferenceFailed, InferenceTimeout, SegmindError

# Async v2: submit and wait (default timeout 600 s). Returns the result dict.
result = segmind.run("p-image", prompt="a red bicycle against a white wall", aspect_ratio="16:9")

# Slow models: keep the handle, wait with your own deadline.
job = segmind.submit_async("<video-model-slug>", prompt="a drone shot over a coastline")
print(job.request_id)  # store it if a worker will collect the result later
try:
    result = job.wait(timeout=900)
except InferenceFailed as e:
    print("generation failed:", e.detail)
except InferenceTimeout:
    print("still running; poll job.status() later")
except SegmindError as e:  # transport errors: 401, 404, 406, 429, 5xx
    print("API error:", e.status, e.detail)

# Sync v1: one blocking call, raw httpx.Response; media bytes in .content.
response = segmind.run_sync("p-image", prompt="a red bicycle")
with open("bicycle.jpg", "wb") as f:
    f.write(response.content)

# Upload local files to get URLs that models accept as inputs.
urls = segmind.files.upload(["photo.jpg"])["file_urls"]

# Pixelflows (saved multi-model workflows).
out = segmind.pixelflows.run(workflow_id="<workflow-id>", data={"prompt": "..."})
```

Use `segmind.SegmindClient(api_key=..., timeout=...)` when the key does not come from the
environment or several clients are needed.

## TypeScript / JavaScript (Node 18+, server side)

```ts
const SEGMIND = "https://api.segmind.com";

function segmindHeaders(): Record<string, string> {
  const key = process.env.SEGMIND_API_KEY;
  if (!key) throw new Error("SEGMIND_API_KEY is not set");
  return { "x-api-key": key, "Content-Type": "application/json" };
}

/** Submit a v2 job and poll until it finishes. Returns the result JSON. */
export async function runSegmind(
  slug: string,
  params: Record<string, unknown>,
  { timeoutMs = 600_000, intervalMs = 2_000 } = {},
): Promise<Record<string, any>> {
  const headers = segmindHeaders();
  const submit = await fetch(`${SEGMIND}/v2/${slug}`, {
    method: "POST",
    headers,
    body: JSON.stringify(params),
  });
  if (!submit.ok) {
    throw new Error(`Segmind submit failed (${submit.status}): ${await submit.text()}`);
  }
  const { request_id, status_url, response_url } = await submit.json();

  const deadline = Date.now() + timeoutMs;
  while (Date.now() < deadline) {
    const res = await fetch(status_url, { headers });
    const body = await res.json().catch(() => ({}));
    if (body.status === "COMPLETED") {
      const result = await fetch(response_url, { headers });
      if (!result.ok) throw new Error(`Segmind result fetch failed (${result.status})`);
      return result.json();
    }
    // A FAILED job arrives as HTTP 422 with the reason in `error`.
    if (body.status === "FAILED") {
      throw new Error(`Segmind request ${request_id} failed: ${body.error ?? "unknown error"}`);
    }
    // Back off on rate limits and server errors; any other non-2xx is a real error.
    if (!res.ok && res.status !== 429 && res.status < 500) {
      throw new Error(`Segmind status check failed (${res.status})`);
    }
    await new Promise((r) => setTimeout(r, intervalMs));
  }
  throw new Error(`Segmind request ${request_id} still running after ${timeoutMs} ms`);
}
```

For a fast image model in a request handler, a single v1 call is simpler:

```ts
const res = await fetch(`${SEGMIND}/v1/p-image`, {
  method: "POST",
  headers: segmindHeaders(),
  body: JSON.stringify({ prompt: "a red bicycle", aspect_ratio: "1:1" }),
});
if (!res.ok) throw new Error(`Segmind error (${res.status}): ${await res.text()}`);
const bytes = Buffer.from(await res.arrayBuffer()); // content type in res.headers
```

## curl

```bash
curl -fsS -X POST "https://api.segmind.com/v2/p-image" \
  -H "x-api-key: $SEGMIND_API_KEY" -H "Content-Type: application/json" \
  -d '{"prompt": "a red bicycle", "aspect_ratio": "1:1"}'
# then: curl -s "<status_url>" -H "x-api-key: $SEGMIND_API_KEY"
#  and: curl -s "<response_url>" -H "x-api-key: $SEGMIND_API_KEY"
```

Full platform docs: https://docs.segmind.com
