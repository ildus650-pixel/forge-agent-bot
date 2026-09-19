# forge-agent-bot

An autonomous coding agent running on GitHub Actions, built on
[Forge](https://github.com/hoangsonww/Forge-Agentic-Coding-CLI).

**Runs at $0 with no API key.** The model runs locally inside the job via Ollama
(`qwen2.5-coder:1.5b`, 986 MB). `cost-usd` for a run reports `0`.

## What the original spec got wrong

The spec conflated two unrelated projects that share the name "Forge", and the
action reference it gave does not resolve. Each point below was verified against
the upstream repositories:

| Spec said | Reality |
|---|---|
| action `.../actions/forge-run@v1` | No `v1` tag exists. Tags are `v0.1.0`, `v0.1.1`, `v1.0.0`, `v1.0.1`. |
| use `@v1.0.1` as a fix | At `v1.0.1` the action file does not exist — the runner fails with `Can't find 'action.yml' ... @v1.0.1`. It exists only on `master`, so `@master` is used. |
| action belongs to `Omar-Azam/forge-agent` | It belongs to `hoangsonww/Forge-Agentic-Coding-CLI`. Different authors, different projects. |
| `docker pull ghcr.io/omar-azam/forge-agent:latest` | The GHCR package rejects anonymous pulls (token request → 403, while a known-public GHCR package returns 200). |
| "no API key, drives free web UIs" | True for Omar-Azam's agent — but it drives DeepSeek/ChatGPT/Gemini through Playwright and halts on an interactive `🔐 LOGIN REQUIRED … press ENTER` prompt read from stdin. There is no unattended CI path and no cookie/`storageState` injection hook. |

So this repo wires the **CI-capable** Forge and uses a local model. Omar-Azam's
agent is not wired, because it cannot run without a human completing a browser
login.

## Why a local model rather than a free hosted one

Z.AI's OpenAI-compatible endpoint does serve the free `glm-4.5-flash`, and Forge
can be pointed at it with `OPENAI_BASE_URL` + `OPENAI_API_KEY`. It was tried and
rejected on evidence:

- first request: HTTP 200 after 9.5 s with an **empty `content`** — the generated
  tokens land in the reasoning channel, so Forge sees no answer;
- retry: no response at all (measured `http=000`), which Forge reports as
  `HeadersTimeoutError` and silently degrades to a **canned fallback plan** —
  a green run that did no work;
- the paid `glm-4.5` is rejected with `429 Insufficient balance`.

A local model removes the external dependency and that entire failure mode.
One provider-name gotcha: the CLI accepts `ollama | anthropic | openai | llamacpp
| vllm | lmstudio`. The README's `openai-compat` is descriptive wording, not a
valid value — passing it fails with `✖ invalid provider=openai-compat`.

## Schedule

| Trigger | When |
|---|---|
| `schedule` | `0 10 * * *` (10:00 UTC daily) |
| `workflow_dispatch` | manual |

```bash
gh workflow run "Forge Agent" --repo ildus650-pixel/forge-agent-bot \
  -f task="Review this repo and list three improvements." -f mode=plan
```

`mode` defaults to `plan` — read-only and safe on any branch. Use `balanced` or
`risky` only if you want Forge to actually write files.

## Required secrets

| Secret | Purpose |
|---|---|
| `TELEGRAM_BOT_TOKEN` | this agent's own bot |
| `TELEGRAM_CHAT_ID` | your chat id for that bot |

No model key is needed.

## Cost and timing

A daily run installs Ollama, pulls the 986 MB model and answers in ~28 s of
inference; the whole job lands around two minutes. Actions minutes on a public
repository are free and unmetered.

## Files

| File | Role |
|---|---|
| `.github/workflows/forge-run.yml` | local model, Forge action, notify, commit |
| `scripts/notify.py` | chunks and posts to Telegram |
| `runs/` | one Markdown file per run |
