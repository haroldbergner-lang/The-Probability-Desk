# The Probability Desk

A multi-agent forecasting dashboard for any question with a genuinely
uncertain outcome — big or small, personal or public ("Will I be married by
30?", "Will she text me back?", or anything else). It is not specific to
prediction markets. It replaces the manual workflow of pasting the same
forecasting prompt into several AI chat tabs and reconciling the answers by
hand: this page sends one standardized prompt to several agent LLMs in
parallel, parses their structured JSON forecasts, computes deterministic
summary stats, runs a synthesis pass on a "host" model, and saves the
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

## Five providers, two roles

There are five providers — Google Gemini, OpenRouter, Groq, Mistral, and
Cohere — and exactly two roles: one **host** and the rest **panel agents**.
Every one of the five was chosen because it can be used for genuinely free,
no-card API access (see the table below); Anthropic, OpenAI, and xAI were
deliberately left out because none of them has any free API tier at all.
SambaNova Cloud was also considered and rejected — it has a generous free
tier, but its API blocks direct browser calls (no CORS headers on
preflight), which would require a backend proxy this app doesn't have.

The host is chosen with a dropdown right on the main page, next to the
question form; every other provider that has a key configured automatically
becomes a panel agent (providers without a key sit idle for manual paste —
see below). The host never forecasts, and switching hosts mid-session
rebalances the panel (the new host's forecast comes out of the active panel,
the old host joins the panel and its agent call runs, and synthesis
re-runs). Nothing is ever discarded — every agent result and every synthesis
pass is kept in the saved record, tagged with the host/panel composition
that produced it.

Default host is Google Gemini, so the default panel is OpenRouter, Groq,
Mistral, and Cohere.

Model IDs move fast, so every provider's model is editable in Settings; the
shipped defaults (current as of August 2026) are chosen to be free out of
the box:

| Provider | Default model | Free tier? |
|---|---|---|
| Google Gemini | `gemini-3.6-flash` | Yes — Flash models are free via Google AI Studio, no card. (The Pro model is not; don't switch to it unless you're willing to pay.) |
| OpenRouter | `openrouter/free` | Yes — no card. This is OpenRouter's own router alias that auto-selects among whatever `:free` models are currently available, so it doesn't go stale the way a specific `:free` model ID does. |
| Groq | `openai/gpt-oss-120b` | Yes — no card, ~30 req/min, 14,400 req/day. (Their Qwen models are "preview" and can be pulled without notice — avoid pinning to one as a default.) |
| Mistral | `mistral-small-latest` | Yes — "Experiment" tier, no card, ~1B tokens/month. (Mistral Large requires their paid Scale plan — Small is what's actually free.) |
| Cohere | `command-a-plus-05-2026` | Yes — no card, but a "trial" key capped at 1,000 calls/month across all endpoints. Fine for occasional personal use; not meant for heavy use. |

None of these five make outbound web-search calls by default. Only Google
Gemini has always-on live search; the other four are answering purely from
training data unless you deliberately enable a paid search add-on (none of
which are free): OpenRouter's "web" plugin toggle in Settings (~$0.007–
$0.015/search) or setting Groq's model to `groq/compound` (~$0.005–$0.008/
search). Every agent card shows a "🔎 Live web search enabled" or "📚 No
live web search" badge reflecting the current configuration, so it's always
visible at a glance which agents can actually look things up.

CORS (calling each API directly from the browser) has been verified for all
five via preflight `OPTIONS` requests — every one returns an
`Access-Control-Allow-Origin` header permitting a browser call, so no
backend proxy is needed for any of them.

## Host synthesis rules

Synthesis runs automatically — there's no button. It fires once at least 2
panel agents have responded (and again if more come in afterward, or if the
panel changes), so long as the host has a working key; there's nothing to
click. Individual agent cards show only the numbers (probability, range,
confidence) by default, with the model's stated reasoning tucked behind a
collapsed "Reasoning" toggle — the final synthesis panel is where the
explanatory detail lives (key driver, main disagreement, and every
adjustment vs. the mechanical average, each with a stated reason).

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

The main page also shows a status chip for every provider (connected / no
key / failed, with the error) right at the top, and re-checks every
configured provider automatically on page load — so you can tell at a
glance which ones are actually ready without opening Settings.

The active run (question, background, every agent's status/result, stats,
and any synthesis passes) is also persisted to `localStorage`, separately
from settings, and restored automatically on page reload — so refreshing
the tab doesn't lose your in-progress forecast. "Clear Form" wipes that
persisted run along with the form fields, for starting a genuinely new
forecast. "Reset Everything" in Settings goes further and erases API
keys, GitHub config, and the current run entirely, back to a blank slate
(with a confirmation prompt, since it can't be undone).

Any subset of the panel agents can fail (bad key, rate limit, CORS) —
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
