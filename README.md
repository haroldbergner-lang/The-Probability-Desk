# The Probability Desk

A multi-agent forecasting dashboard for any question with a genuinely
uncertain outcome — big or small, personal or public ("Will I be married by
30?", "Will she text me back?", or anything else). It is not specific to
prediction markets. It replaces the manual workflow of pasting the same
forecasting prompt into several AI chat tabs and reconciling the answers by
hand: this page sends one standardized prompt to four agent LLMs in
parallel, parses their structured JSON forecasts, computes deterministic
summary stats, runs a synthesis pass on a fifth "host" model, and saves the
result to this repo as a scoreable forecast log.

Pure client-side, no build step, no framework, no external CDN dependencies.
Everything lives in two self-contained HTML files.

## Pages

- **`index.html`** — set up a question, assemble the standardized prompt, run
  it against the panel of agents, review stats and the host's synthesis, save
  the forecast to `forecasts/`.
- **`forecasts.html`** — the forecast log: a sortable table of every saved
  forecast, YES/NO resolution marking, and a running Brier score over resolved
  forecasts.

## The five providers, two roles

There are exactly five providers — Anthropic, OpenAI, Google Gemini, xAI
(Grok), and OpenRouter — and exactly two roles: one **host** and four **panel
agents**. The host is chosen with a dropdown right on the main page, next to
the question form; the other four providers automatically become the panel.
The host never forecasts, and switching hosts mid-session rebalances the
panel (the new host's forecast comes out of the active panel, the old host
joins the panel and its agent call runs, and synthesis re-runs). Nothing is
ever discarded — every agent result and every synthesis pass is kept in the
saved record, tagged with the host/panel composition that produced it.

Default host is Anthropic, so the default panel is OpenAI, Gemini, xAI, and
OpenRouter.

Model IDs move fast, so every provider's model is editable in Settings; the
shipped defaults (current as of August 2026) are:

| Provider | Default model | Web grounding |
|---|---|---|
| Anthropic | `claude-opus-5` | `web_search` server tool |
| OpenAI | `gpt-5.1` | Responses API `web_search` tool |
| Google Gemini | `gemini-3.1-pro-preview` | `google_search` grounding tool |
| xAI (Grok) | `grok-4.6` | Live Search (`search_parameters`) |
| OpenRouter | `deepseek/deepseek-v4-pro` | optional `web` plugin (toggle in Settings) |

OpenRouter defaults to DeepSeek V4 Pro as a strong, distinct-family model that
isn't Claude/GPT/Gemini/Grok-derived — swap it for any OpenRouter model string.

xAI's browser CORS support is poorly documented and appears to vary. If a
direct call to `api.x.ai` fails with what looks like a CORS/preflight
rejection, the app automatically retries that agent by routing it through
OpenRouter instead (configurable fallback model, default `x-ai/grok-4.6`) —
provided an OpenRouter key is configured.

## Host synthesis rules

The host never forecasts — it starts from the mechanical average of the
panel and adjusts only for stated reasons: agreement vs. disagreement among
the agents, the strength of each agent's reasoning, and outliers. It never
overweights one agent without a concrete reason. It widens the uncertainty
range when the panel is dispersed, and raises its confidence rating when the
panel is tightly clustered with sound reasoning (lowering it when dispersed
or the reasoning is weak) — calibrated realism over false precision.

## Bring your own keys

All API keys and the GitHub token are entered in the Settings modal and
stored only in this browser's `localStorage`. They are never written to the
repo and never logged. Every provider row in Settings has a cheap "Test
connection" call so you can confirm a key works before running a real
forecast.

Any subset of the four panel agents can fail (bad key, rate limit, CORS) —
the rest of the run still completes (`Promise.allSettled`), and every agent
card has a manual-paste fallback: run that model by hand in a chat tab and
paste its JSON into the card to have it join the panel identically to an API
call.

## Forecast log format

`Save Forecast` commits `forecasts/YYYY-MM-DD-slug.json` via the GitHub
contents API, using a fine-grained, repo-scoped personal access token. Each
record contains the question, optional background, the host/panel
composition, every agent's raw and parsed output (including which were
manually pasted), deterministic stats, every synthesis pass (each stamped
with its own host and panel), and — once you mark it on `forecasts.html` —
a resolution.

## Local preview

No build step is needed; just serve the two files statically, e.g.:

```
python3 -m http.server 8000
```

then open `http://localhost:8000/index.html`.
