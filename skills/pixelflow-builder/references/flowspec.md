# FlowSpec: writing a flow

Contents: shape · keys · references · params · outputs · limits · validator codes

The MCP tools take a **FlowSpec**, never raw graph JSON. Segmind compiles it into the editor's graph and decompiles saved graphs back into one.

```json
{
  "name": "Product ad pipeline",
  "inputs":  [{"key": "product_image", "kind": "image"},
              {"key": "brief", "kind": "text"}],
  "steps":   [{"id": "cutout", "model": "bg-removal-v2",
               "params": {"image": "{{ product_image }}"}},
              {"id": "scene", "model": "flux-kontext-pro",
               "params": {"input_image": "{{ cutout }}",
                          "prompt": "Studio ad shot. {{ brief }}"}}],
  "outputs": [{"key": "ad_image", "from": "scene"}]
}
```

## Shape

- `name`: human-readable, at most 120 chars. Ignored by `update_pixelflow` (renaming happens in the editor).
- `inputs[]`: `{key, kind, example?, required?}`. `kind` is `image | video | audio | file` (takes an https URL) or `text | number` (takes a literal). `required` defaults to true and is informational only.
- `steps[]`: `{id, model, params}`. `model` is a catalog slug as `list_models` shows it. Steps may be listed in any order. They run by dependency.
- `outputs[]`: `{key, from}`. `from` is `<step id>` or `<step id>.<output key>`, never an input.
- No other fields anywhere (`SPEC_MALFORMED`).

## Keys and ids

- Inputs, step ids and output keys share **one namespace**. Use each name once (`DUPLICATE_KEY`). For example, don't call both an input and a step `image`.
- Pattern: lowercase letters, digits and single `_`, starting with a letter and not ending in `_`, at most 40 chars. No `__`. Not `item`, `index`, `true`, `false`, `null`, `undefined` or `this` (`INVALID_KEY`).
- Use names that say what the thing is: `product_image`, `brief`, `headshot`, `ad_video`. Not `Output`, `img1` or `Image_Output`.

## References

Only three forms are allowed inside `{{ }}`:

| Form | Means |
|---|---|
| `{{ product_image }}` | the value of an input |
| `{{ cutout }}` | step `cutout`'s primary output |
| `{{ clip.last_frame }}` | a named output of step `clip` |

- No indexes (`[0]`), deeper paths or expressions (`UNSUPPORTED_REF`). The compiler adds `[0]` wherever a list feeds a single-value field.
- References go **only inside string values**, and can sit inside text: `"Studio ad, {{ brief }}"`.
- **List parameters** (`image_urls`, `reference_images`) take several references as **one string**: `"{{ photo_a }} {{ photo_b }}"`. Never write a JSON list of references.
- References separated only by spaces become a list. Into a single-value field such as `prompt`, that is `ARRAY_INTO_SCALAR`. Put words or punctuation between them: `"{{ style }}, {{ subject }}"`.
- A named output key must be one the model really produces. `get_model_spec` and the validator hint list them. Image models usually output `images`, video models `video`, audio models `audio`, LLMs `text`. Some outputs are optional and need a switch (seedance's `last_frame` needs `return_last_frame: true`). For an LLM step, `{{ step.key }}` reads that key of its JSON reply.

## Params

- Use the model's own parameter names from `get_model_spec` (`UNKNOWN_PARAM` lists the real ones). Set every required one (`MISSING_REQUIRED_PARAM`).
- Enum params take one of the listed options exactly (`"9:16"`, `"2x"`, `"720p"`). Don't invent values.
- One field holds either a literal or a reference, never both.
- Every piece of the user's media is an `input`. Never paste a URL into `params`. Fixed assets that belong to the flow itself, like a brand logo every run uses, are the exception.
- A literal numeric `seed` locks the seed. Leave it out so each run varies, unless the user wants reproducible output.
- Price drivers (`duration`, `resolution`, `upscale_factor`, number of outputs) are set deliberately. Video steps dominate cost.
- `example` on a media input must be a long-lived http(s) URL. Never use an upload URL (`create_upload_ticket` / `upload_media` URLs expire in about two days), and never a local path.

## Outputs

- At least one output, at most 10 (`NO_OUTPUT`, `TOO_MANY_OUTPUTS`).
- One output per deliverable, with a key that says what it is (`ad_video`, `headshot_image`).
- Every step should lead to an output. A step nothing reads from is wasted spend (`UNREACHABLE_STEP` warning).

## Limits and what a flow cannot do

- At most 20 steps, 10 inputs and 10 outputs, and 1 MB compiled (`TOO_MANY_STEPS`, `TOO_MANY_INPUTS`, `GRAPH_TOO_LARGE`).
- No branches, conditions, loops, fan-out or editor-only control nodes. Utility models such as `video-audio-merge` are ordinary steps. "For each of N" means running the flow N times. "If A then B" means deciding in chat and saving the chosen path, or making the choice an input (a text `aspect_ratio` input fed into the model's `aspect_ratio` param).
- Flows built in the Pixelflow editor usually decompile with opaque nodes (`editable: false`). They are read-only from the MCP and must be changed in the editor.

## Validator codes

`validate_pixelflow` returns every problem at once as `{code, path, message, hint}`. `path` is in FlowSpec terms (`steps[1].params.first_frame_url`). The hint usually carries the fix. Fix all of them, then validate again.

| Code | Cause | Fix |
|---|---|---|
| `SPEC_MALFORMED` | Unknown field, or a bad `example` (media not http(s), number not a number) | Remove the field or fix the example |
| `INVALID_KEY` | Key breaks the pattern (`Out_Video`, `photo__a`, `item`) | Rename to lowercase snake case |
| `DUPLICATE_KEY` | Same name used twice across inputs, steps and outputs | Rename one |
| `NO_OUTPUT` / `TOO_MANY_*` / `GRAPH_TOO_LARGE` | Size rules | Add an output, or cut steps |
| `UNKNOWN_MODEL` / `MODEL_NOT_RUNNABLE` | Slug not in the catalog, or not live | Pick a live slug with `list_models` |
| `MODEL_LOOKUP_FAILED` | Catalog unreachable (transient) | Validate again, unchanged |
| `UNKNOWN_PARAM` | Parameter name the model does not have | Use a name from the hint |
| `MISSING_REQUIRED_PARAM` | A required parameter is unset | Set it or wire an input to it |
| `INVALID_OPTION` | A value outside the parameter's listed options | Use one of the options exactly |
| `UNKNOWN_REF` | `{{ x }}` names no input or step, or an output reads an input | Fix the name; outputs read steps |
| `UNKNOWN_OUTPUT_KEY` | `{{ step.key }}` where the model has no such output (`{{ edit.image }}` for a model that outputs `images`) | Use `{{ step }}` or a key from the hint |
| `UNSUPPORTED_REF` | Reference inside a JSON list or object, or an index or path (`{{ a.images[0] }}`) | One string `"{{ a }} {{ b }}"`; drop the index |
| `ARRAY_INTO_SCALAR` | `"{{ a }} {{ b }}"` into a single-value field | Put text between them, or keep one |
| `HANDLE_KIND_MISMATCH` | Wrong media kind wired (text into an image param, video into an image param) | Wire a step of the right kind |
| `CYCLE` | A step depends on itself, directly or through others | Re-order the chain |
| `OPAQUE_NODES` (on update) | Editor-only nodes | Change it in the editor |

Warnings don't block saving, but tell the user: `NO_SCHEMA` (a model without a published schema, so parameter names were not checked), `COST_UNKNOWN` (a step priced per second or per token), `UNREACHABLE_STEP` (a step no output reads), `UNUSED_INPUT` (an input nothing reads), `HARDCODED_URL` (a URL pasted into params that should be an input).
