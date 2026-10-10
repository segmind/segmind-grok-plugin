# Worked examples

Contents: 1 product ad pipeline (build) · 2 batch on local files (build, publish, run) · 3 edit and undo · 4 save what we just did · 5 one-off (don't build)

Every FlowSpec here passed `validate_pixelflow`. Model slugs and parameters change, so still call `get_model_spec` for each model before reusing a step.

## 1. Product ad pipeline

> "Make me a reusable workflow: I give it a product photo and a one-line brief, and it gives me back a 5-second vertical ad video with a voice-over reading the brief."

The user asked to save it, so build. They asked for no run.

1. `list_pixelflows(search="product video")`, then `get_pixelflow_spec` on the best hit. If nothing passes the checklist (inputs: product image + brief text; output: video with voice; `runnable`), build.
2. Shortlist with `list_models`: image-to-video `seedance-2.0-fast` (takes the product as its first frame), voice `tts-eleven-labs`, and the `video-audio-merge` utility to lay the voice over the clip. Call `get_model_spec` for each one.
3. Validate, show the plan and cost, then create:

```json
{
  "name": "Product ad video with voice-over",
  "inputs": [
    {"key": "product_image", "kind": "image"},
    {"key": "brief", "kind": "text"}
  ],
  "steps": [
    {"id": "clip", "model": "seedance-2.0-fast",
     "params": {"first_frame_url": "{{ product_image }}",
                "prompt": "Slow cinematic push-in on the product, studio light, premium ad. {{ brief }}",
                "duration": 5, "aspect_ratio": "9:16", "resolution": "720p", "generate_audio": false}},
    {"id": "voice", "model": "tts-eleven-labs",
     "params": {"prompt": "{{ brief }}", "voice": "Rachel", "model_id": "eleven_multilingual_v2"}},
    {"id": "ad", "model": "video-audio-merge",
     "params": {"input_video": "{{ clip }}", "input_audio": "{{ voice }}"}}
  ],
  "outputs": [{"key": "ad_video", "from": "ad"}]
}
```

4. Reply: what each step does, the per-run cost (or "at least $X" when a step is unpriced), and the `editor_url`. End there. **Don't publish and don't run.** Offer: "Want me to publish it as an API, or run it on a product photo?"

## 2. Batch on local files

> "Turn these 6 selfies in ~/team/ into clean corporate headshots, upscaled. We do this every month for new hires, so set it up properly."

"Set it up properly" plus a monthly repeat means build. "Turn these 6 into..." means run it now. Running a saved flow needs a published version, so publishing is part of the request.

1. Build and validate:

```json
{
  "name": "Corporate headshot",
  "inputs": [{"key": "selfie", "kind": "image"}],
  "steps": [
    {"id": "headshot", "model": "nano-banana-2",
     "params": {"image_urls": "{{ selfie }}",
                "prompt": "Turn this selfie into a clean corporate headshot: same person and face, business attire, neutral grey studio backdrop, soft key light, head and shoulders, sharp focus.",
                "aspect_ratio": "4:5"}},
    {"id": "upscaled", "model": "topaz-image-upscale",
     "params": {"image": "{{ headshot }}", "upscale_factor": "2x", "enhance_model": "Standard V2"}}
  ],
  "outputs": [{"key": "headshot_image", "from": "upscaled"}]
}
```

2. `create_pixelflow`.
3. Say in one line: "To run it I'll publish it as an API version. That version is unlisted (anyone with its slug could run it on their own credits) and permanent." Then `publish_pixelflow` once and keep the `api_slug`.
4. Upload: for each file, `stat -f%z ~/team/alice.jpg` → `create_upload_ticket("alice.jpg", <size>)` → `curl` PUT with the required headers → `url_after_upload`.
5. `get_pixelflow_spec(api_slug)` gives the keyname `selfie`. State the total: about 6 x $0.48.
6. Run the first one, check it, and look at the result. Then submit the other 5 in waves of at most 3, calling `check_pixelflow` until each finishes.
7. Report one line per person, with the output URL. Next month the user only needs the upload and submit steps with the same `api_slug`.

Wrong ways to do this: running `nano-banana-2` and then `topaz-image-upscale` directly for each selfie after saving the flow. That is the same cost with none of the reuse, and the saved flow goes untested. Also wrong: a flow with 6 image inputs, or any attempt at fan-out.

## 3. Edit and undo

> "In my headshot flow, swap the upscaler for something cheaper."

1. `list_pixelflows(source="mine", search="headshot")` → `graph_id`.
2. `get_pixelflow(graph_id)` → `flow`, `version: 1`, `editable: true`. If `editable` were false, the answer would be the `editor_url` and nothing else.
3. `compare_models(["topaz-image-upscale", "clarityai-crystal-upscaler", ...], scenario=...)` to show the price difference, then `get_model_spec("clarityai-crystal-upscaler")`.
4. Change **only** the upscale step. The prompt, aspect ratio, inputs and outputs stay exactly as read:

```json
{"id": "upscaled", "model": "clarityai-crystal-upscaler",
 "params": {"image": "{{ headshot }}", "scale_factor": 2}}
```

5. `validate_pixelflow(full flow)` → `update_pixelflow(graph_id, version=1, flow=...)` → result `version: 2, previous_revision: 1`. Tell the user the new per-run cost, and that the published API version (if any) is unchanged until they ask to publish again.
6. "Undo that" → `restore_pixelflow(graph_id, revision=1)`. The result names a new `previous_revision`, so the restore can be undone the same way.
7. A `409` on update means someone changed the flow (perhaps in the editor) after it was read. Read it again, re-apply the swap to the fresh flow, confirm, and save with the new `version`.

## 4. "Save what we just did"

Earlier in the chat: `bg-removal-v2` on the user's bottle photo, then `flux-kontext-pro` placing the cut-out on a marble counter, then `seedance-2.0-fast` animating it. Now: "Love it. Save that as a workflow so I can run it on my other products."

Compile the steps that worked into a FlowSpec. The user's photo becomes an input; it is not the URL from history. Each later step reads the earlier one by reference. Prompts and settings carry over unchanged.

```json
{
  "name": "Product bottle to marble push-in video",
  "inputs": [{"key": "product_image", "kind": "image"}],
  "steps": [
    {"id": "cutout", "model": "bg-removal-v2", "params": {"image": "{{ product_image }}"}},
    {"id": "scene", "model": "flux-kontext-pro",
     "params": {"input_image": "{{ cutout }}",
                "prompt": "Place this product on a marble kitchen counter, soft morning light, product photography"}},
    {"id": "video", "model": "seedance-2.0-fast",
     "params": {"first_frame_url": "{{ scene }}", "prompt": "slow camera push-in, gentle light flicker", "duration": 5}}
  ],
  "outputs": [{"key": "product_video", "from": "video"}]
}
```

Validate, create, and give the user the `editor_url`. Don't publish or run. Steps that failed or were thrown away in the session don't go in. If the session used a model that is no longer live, `validate_pixelflow` says so: swap in its newest sibling and tell the user.

## 5. One-off: don't build

> "Generate an image of a cat astronaut floating above Earth, then animate it into a short video. Just this once."

Compose: run the text-to-image model, then pass its output URL as the first frame of an image-to-video model with `submit_inference` and `check_inference`. No `validate_pixelflow`, no `create_pixelflow`. If the user likes the result, offer once to save it as a flow.
