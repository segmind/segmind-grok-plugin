# Segmind plugin for Grok Build

Connect Grok Build to [Segmind](https://www.segmind.com): find, price and run 500+ image,
video, audio, 3D and language models, find, build and run Pixelflows (saved multi-model
workflows), and save the results straight into your project.

The plugin bundles the hosted Segmind MCP server and three skills. It installs nothing on
your machine and ships no hooks or scripts.

| Component | What it does |
| --- | --- |
| MCP server `segmind` | Hosted at `https://mcp.segmind.com/mcp` (Streamable HTTP). |
| Skill `segmind` | Pick a model, price it, run it, and save the output into the repo. Find past runs and outputs. |
| Skill `pixelflow-builder` | Find, run, build, edit, undo and publish Pixelflows. |
| Skill `segmind-api` | Write app code that calls the Segmind API (Python SDK, TypeScript, curl). |

`pixelflow-builder` is maintained by Segmind alongside the MCP server and copied here
unchanged; update it there, not in this repo.

## Install

In Grok Build, open `/plugins`, find **Segmind** in the Marketplace tab, and install it.

To install straight from this repo, or from a local checkout, instead:

```bash
grok plugin install segmind/segmind-grok-plugin
```

```bash
grok plugin install /path/to/segmind-grok-plugin
```

## Authentication

The plugin uses OAuth. On the first tool call, Grok opens Segmind's sign-in page in your
browser. Sign in with your Segmind account and approve the connection. You can also start
it from `/mcps`: select `segmind` and press `i`.

- No API key is stored in the plugin, and you never paste one into chat.
- Runs bill the Segmind account you approved at sign-in. Ask "how many Segmind credits do
  I have?" to see the balance and which account is billed. To switch accounts, reconnect
  and pick another account on the sign-in screen.

## Try it

- "Generate a 16:9 hero image of a lighthouse at dusk with Segmind and save it to
  `public/hero.jpg`."
- "Which Segmind text-to-video model is cheapest for a 5-second 1080p clip?"
- "Upscale `assets/logo-draft.png` with Segmind."
- "Save this product-photo-to-ad-video chain as a Segmind Pixelflow and run it on
  everything in `assets/products/`."
- "Add a `/api/generate` route that calls Segmind's p-image model."

## Network endpoints

| Endpoint | Why |
| --- | --- |
| `https://mcp.segmind.com/mcp` | The hosted MCP server. Every tool call goes here. |
| `https://cloud.segmind.com/api` | OAuth 2.1 authorization server: discovery, `/oauth/authorize`, `/oauth/token`. |
| `https://platform.segmind.com` | The browser sign-in and consent pages. |
| `https://segmind-transient-inputs.s3.amazonaws.com` | Presigned upload URLs. Used only when you ask to use a local file as a model input; the agent PUTs that one file. |
| `https://images.segmind.com` | Where generated outputs and uploaded inputs are served from; the agent downloads outputs from here into your project. |

The `segmind-api` skill writes code that calls `https://api.segmind.com` with an API key
your app reads from its own `SEGMIND_API_KEY` environment variable. The skill tells the
agent never to read your `.env` or other secret files, hard-code a key, or commit one.

## Credentials and permissions

- **Credential:** a Segmind account, used via OAuth. The consent screen lists the scopes
  Segmind's authorization server offers: `models:read`, `models:invoke` (running models,
  which spends credits), `workflows:read`, `workflows:write`, `profile`, `email` and
  `offline_access` (so the connection stays signed in).
- **Spending:** generation and Pixelflow runs bill your Segmind credits, within your
  account's spend limit. The skills tell the agent to state the cost before any run over
  $0.50 and the total before a batch, and never to start a run your stated budget cannot
  cover.
- **Pixelflows:** `workflows:write` lets the agent save and edit flows in your account. It
  publishes a flow only when you ask to publish or run it. A published version is
  unlisted, not private (anyone with its slug can run it on their own credits), and
  permanent.
- **Local files:** read only when you ask to use one as an input (to upload it) or when
  saving outputs to the path you chose.

## Support

- Docs: https://docs.segmind.com
- MCP setup guide: https://www.segmind.com/mcp
- Issues: https://github.com/segmind/segmind-grok-plugin/issues

## License

MIT. See [LICENSE](LICENSE). Use of the Segmind service is governed by Segmind's terms.
