# Running a Pixelflow

Contents: what can run · the spec · local files · submit and check · outputs · failures · batches

## What can run

- A **catalog flow**: its `slug` from `list_pixelflows`, e.g. `ai-video-enhancer`.
- A **published version of the user's own flow**: its `api_slug` `<graph_id>-vN`, or the bare `graph_id` for the latest published version. A flow that was saved but never published cannot run. See SKILL.md §5 for when publishing is allowed.

## 1. Read the spec

`get_pixelflow_spec(slug)` is required before every first run of a flow. Input keynames are arbitrary names chosen when the flow was built (`3Input_images__`, `video_in`, `image_kj3ba`) and cannot be guessed.

| Field | Use |
|---|---|
| `inputs[].keyname` | The key to send |
| `inputs[].kind` | `image` / `video` / `audio` take an https URL; `text`, `integer`, `number` take a literal; `file` means media of a kind the flow does not say |
| `inputs[].feeds` | Which model parameter the input reaches. This tells you what the input really is |
| `inputs[].is_list` | true: send several URLs (a JSON list) |
| `inputs[].example` / `required` | Saved sample value; whether a run must send it (own flows) |
| `inputs_skeleton` | A ready-to-fill `inputs` object. Replace every placeholder |
| `runnable` / `not_runnable_reason` | false: don't run it |
| `models`, `nodes` | The chain, and `node_id` → model, for reading failures |
| `estimated_cost` | `min`/`max` for one run, or null with `known_min` + `unpriced_steps`. Pass the planned `inputs` to price input-dependent steps (duration, resolution) |

## 2. Local files

Media inputs must be URLs the provider can fetch. A local path is refused before billing.

- Claude Code, the Segmind Agent and other hosts that can read the file: `create_upload_ticket(filename, size_bytes)` with the size **measured** (`stat -f%z file` on macOS, `stat -c%s file` on Linux). PUT the bytes with every header in `required_headers`, then use `url_after_upload`. Too large a size fails. Too small a size silently stores a truncated file.
- ChatGPT, where the host attaches the file to the tool call: `upload_media(file=...)`.
- claude.ai and other hosts without a shell: ask the user for the file size, give them the `curl` command from the ticket, and wait for them to confirm. Or ask for a link.
- Outputs from earlier runs are already hosted URLs. Pass them straight in with no upload.
- Upload URLs expire in about two days. Use them for runs, never as a saved `example`.

## 3. Submit, then check in the same reply

```
submit_pixelflow(slug="<slug or api_slug>", inputs={"<keyname>": "https://...", ...})
check_pixelflow(request_id="...")   # waits up to ~45 s on the server
check_pixelflow(request_id="...")   # call again at once while pending: true
```

- `inputs` is a flat object keyed by keyname. An unknown key, or a missing media input with no saved value, is refused before anything is billed.
- There is no blocking run. Every flow goes through submit then check.
- Keep calling `check_pixelflow` in the same reply until the run finishes, then give the user the result. Don't end the turn on a queued run. Don't pause between checks, because the server already waits. After about 15 minutes of pending, give the user the `request_id` and say it is still running.
- Never resubmit a run that is still pending. That bills twice.

## 4. Outputs

- Outputs come back keyed by the flow's output keyname, usually as `https://images.segmind.com/...` URLs that can feed another model or flow directly.
- A run that reports success with an empty output is a flow bug. Report it. Don't rerun it, because a rerun bills again for the same empty result.
- Show the user what came back. Look at images, and check a video's length and aspect ratio when you can, before saying it worked.

## 5. Failures

- A failure names the failing node id and its error, plus any outputs from branches that finished (those steps billed). Map the node to its model with the spec's `nodes`.
- `image not found`, or a bare status code, from a node whose inputs are right means that node's model is broken. Report it. Don't retry, and don't hunt for a bad input.
- A failed catalog flow: compose the same chain from live models instead, and say so.
- A failed own flow: fix the step with `update_pixelflow` (swap the model, fix the param), validate, publish a new version **only if the user agrees**, then rerun.
- Insufficient credits, or the account's spend limit reached: stop, say so, and don't retry. The billed account is switched on Segmind, not from a tool.

## 6. Batches

1. Get the spec once.
2. Upload all local files first.
3. State the total: N x the per-run cost.
4. Run the first input, check it, and look at the result. If it is wrong, fix the flow before spending on the rest.
5. Submit the rest in waves of at most 3 concurrent runs. Wider bursts trip provider rate limits. Check each wave to completion before starting the next.
6. Report one line per input: its name, the output URL, or the error.
