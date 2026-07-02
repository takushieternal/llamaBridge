# 🦙 Llama Bridge

**Let Claude hand off simple jobs to free AI models running on your own computer — and see the receipts.**

Llama Bridge is a single-file, pure-Python tool (no dependencies beyond Python 3.8+) that connects [Claude](https://claude.ai) to local [Ollama](https://ollama.com) models over MCP. Claude keeps the hard thinking; routine chores — summarizing, classifying, extracting, reformatting, drafting, simple code — run locally for free. Every handoff is checked, logged to SQLite, and measured so you know exactly what you saved.

```
Claude  →  Llama Bridge  →  Ollama (local, free)  →  checks  →  Claude
```

## Why

- 💰 **Save money.** Cloud AI charges per token it writes. Local models write for free. Savings are counted honestly: the output price you skip, *minus* the input price Claude pays to read the answer back.
- 🏠 **Keep data at home.** Delegated work never leaves your machine.
- ✅ **Trust, but verify.** Automatic checks on every result; failed checks retry on a bigger model; anything still failing is flagged `[UNVERIFIED]` so Claude redoes it.
- 📊 **See the receipts.** A live dashboard logs every delegation: prompt, response, latency, tokens saved, cost avoided, and a per-model scoreboard.

## Quick start

**1. Install Ollama and pull a starter model**

```bash
ollama pull llama3.2:3b
```

**2. Get the MCP config**

```bash
python llama_bridge.py install
```

Paste the printed JSON into your Claude MCP config (e.g. `claude_desktop_config.json`), or run the printed `claude mcp add` command for Claude Code. Restart Claude.

**3. Run the dashboard**

```bash
python llama_bridge.py
```

Opens at `http://127.0.0.1:8765`. The dashboard and the MCP server share one SQLite database, so you see Claude's delegations live. Ask Claude to "check ollama_status" to confirm the hookup.

> Delegation is an optimization, never a dependency. If Ollama is off, Claude just does the work itself.

## The tools Claude gets

| Tool | Purpose |
|---|---|
| `delegate_task` | Run a prompt on a local model. `model` optional — the bridge picks family-aware. |
| `delegate_async` / `delegate_result` | Background jobs for big, slow models (no tool timeouts). |
| `list_models` | Installed models with size and parameter count. |
| `recommend_model` | Preview the auto-pick for a described task + budget. |
| `model_scoreboard` | Per model × task type: uses, ratings, verified %, tok/s. |
| `warm_model` | Preload a model to skip cold-start latency. |
| `ollama_status` | Is Ollama up? Version + loaded models. |
| `get_stats` | Delegations, tokens offloaded, net cost avoided, cache hits, latency. |

## Verification — the safety net

Pass checks with any delegation:

- `require_json` — output must parse as JSON (also enables Ollama JSON mode)
- `must_include` (+ `ignore_case`) — required substrings
- `min_length` — minimum output size
- `escalate_to` — on a failed check, retry once on a bigger model

Two checks run on **every** delegation automatically: empty output, and **likely prompt truncation** (the model evaluated far fewer prompt tokens than the prompt contains — i.e. the context window silently clipped your input). Both trigger escalation. Results that still fail come back tagged `[UNVERIFIED on <model>: <reasons>]` — don't ship those.

## Which models to install

Golden rule: match the model's **family** to the job first, then pick the smallest **size** that gets it done. The auto-picker follows the same rule (code → coder, vision → vision, everything else → general).

| Tier | Example | Best at |
|---|---|---|
| ~3B general | `llama3.2:3b` | High-volume trivial chores: labels, tiny reformats. Start here. |
| ~7–8B general | `qwen2.5:7b` | Everyday summaries, drafts, extraction, simple JSON. |
| ~12B general | `gemma` 12B-class | Broader knowledge, multi-step instructions, cleaner JSON. |
| ~30B coder | `qwen3-coder:30b` | Real code, hardest jobs. Slow — use `delegate_async`. |
| Vision | `qwen3-vl`, `llama3.2-vision` | Images and screenshots. Text-only models can't see. |

A good starter set: one small general + one mid general + whichever specialist matches your work. RAM needed ≈ model file size plus headroom.

## Extras

- **Response cache** — identical verified requests return instantly and free; self-prunes (30-day TTL, 1000-entry cap). Pass `no_cache: true` for creative/nondeterministic tasks.
- **Scoreboard learning** — rate results 👍/👎 in the dashboard; the picker prefers models with a real track record (ratings need ≥3 votes before they count, verified-rate breaks ties, cache hits excluded).
- **Auto escalation ladder** — when the bridge auto-selects, it pre-fills `escalate_to` with the next-size-up same-family model.
- **Hardened dashboard** — API only answers localhost Host/Origin requests (DNS-rebinding/CSRF protected); binds to 127.0.0.1 only.

## Field test

This README's sibling [explainer page](llama-bridge-explainer.html) was built as a live test: two drafting jobs delegated through the bridge (`qwen2.5:7b` + `llama3.2:3b`), both verified on the first try, 694 tokens offloaded, ~6.6 s average round trip, zero failures. Local models draft; Claude judges.

## CLI

```
python llama_bridge.py            # dashboard (GUI + REST API)
python llama_bridge.py mcp        # MCP stdio server (what Claude launches)
python llama_bridge.py install   # print MCP config to paste into Claude

  --port N            dashboard port (default 8765)
  --ollama-host URL   override Ollama base URL (default http://127.0.0.1:11434)
  --db PATH           path to the log database
  --no-browser        don't auto-open the dashboard
```

## Changelog

**v1.4**
- Family-aware model auto-selection (coder models no longer picked for knowledge tasks)
- Rating gate (≥3 votes) + shrinkage; verified-rate tiebreak; cache hits excluded from scores
- Auto `escalate_to` on auto-select (next size up, same family)
- Always-on checks: empty output + likely prompt truncation
- Net cost accounting: `price_out − price_in`
- Localhost Host/Origin enforcement on the dashboard API; quote-safe HTML escaping
- Response cache TTL + size cap

**v1.3**
- Response cache, background jobs, model scoreboard, ratings, cost tracking, streaming test panel

## Requirements

- Python 3.8+ (standard library only)
- [Ollama](https://ollama.com) running locally with at least one model pulled

---

*One Python file, zero dependencies, receipts included.* 🦙
