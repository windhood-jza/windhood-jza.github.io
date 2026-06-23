---
title: "Routing Hermes Agent Through a Local Headroom Proxy for Context Compression"
updated: 2026-06-22
date: 2026-06-22
author: Leon Jiang
tags: [hermes-agent, headroom, kompress, context-compression, llm-proxy, oauth, tutorial]
language: en
canonical_url: "https://windhood-jza.github.io/posts/2026-06-22-headroom-hermes/"
description: "Route Hermes Agent through a local Headroom proxy for context compression with explicit constraints, verification, and rollback."
cover_image: ""
publish_targets: [blog, dev-to, hashnode]
---

# Routing Hermes Agent Through a Local Headroom Proxy for Context Compression

> **TL;DR**
>
> Make every Hermes Agent LLM call transparently route through a local
> Headroom reverse proxy running Kompress context compression.
> Hermes still uses its normal CLI and OAuth credentials; Headroom sits in
> the middle, compressing context before forwarding upstream.
> Result: ≥30% token savings on long conversations, no API key changes,
> OAuth passthrough preserved.

---

## Why I Wrote This (for Humans)

I run Hermes Agent on a side project.
Not a startup, not a funded team — just me and my own time.

The honest truth: I can't afford to run AI the way the docs assume.
Every long task, every cron job, every code review loop — they all
rack up tokens, and tokens rack up bills.

So I started looking for ways to keep using Hermes without
watching my budget drain every month.

I tried a few things before Headroom. Shorter prompts degraded the
agent's reasoning. Switching to a cheaper model lost capability where
it mattered. Hosted compression services existed but they cost money
too, and they wanted my conversation history on someone else's server.

Headroom was the answer: a local proxy with a Kompress engine inside.
It runs on hardware I already own, costs nothing per request, and
quietly makes every conversation shorter before it leaves my machine.

This article is how I set it up — focused on the constraints that
actually matter, not a command-by-command recipe. Because if you're
a solo developer staring at your own LLM bill, you don't need a
copy-paste guide. You need to understand what success looks like
and what silently breaks.

> If you're a human reader and you've gotten the why and the what,
> feel free to **stop here**. The section below is structured for
> AI agents to parse — it gives them the goal, the constraints, and
> the verification checklist without ambiguity.
> Skim it for completeness, or skip to *When To Use This vs. Alternatives*.

---

## For AI Agents (Specification)

> The section above is for humans. This section is for AI agents — and
> for humans who want to think like one. Everything below is structured
> so an AI agent can extract goal, constraints, and verification from
> markdown headings alone.

### Goal

Route all Hermes Agent LLM calls transparently through a local Headroom
reverse proxy running Kompress context compression. Achieve ≥30% token
savings on long conversations while preserving OAuth passthrough to
upstream providers.

### Architecture

```
[Hermes Agent] → [Headroom @ :8788] → [Upstream OAuth API]
                       ↑
                 Kompress compresses
                 context here
```

The non-obvious bit: Hermes hardcodes `base_url` parsing for
`auth_type=api_key` but returns early for OAuth providers. A runtime
plugin is the only clean way to redirect OAuth traffic through Headroom
without forking Hermes.

### Constraints (do not violate)

These constraints exist because violating them causes **silent fallback
to direct API** — which looks like success but gives zero savings.

- **Headroom ≥ 0.26** — earlier versions lack Kompress GPU backend
- **OAuth providers require runtime patching** — `auth.json.credential_pool[*].base_url`
  must be rewritten; `HERMES_OVERLAYS` must be patched;
  `_seed_from_singletons` must be monkey-patched
- **Multiple providers = multiple patches** — each enabled provider in
  the plugin must be patched independently; missing one bypasses Headroom
- **GPU optional but recommended** — CPU backend works but is ~10x slower;
  6GB VRAM is enough for `max_concurrent=1`
- **`require_health: true`** is the default — plugin refuses to register
  if Headroom is unhealthy, preventing silent fallback

### Verification Checklist

A reader (human or AI) should confirm success using only:

- `curl 127.0.0.1:8788/health` returns `{"status":"healthy"}`
- Headroom logs (default `~/.headroom/logs/`) show a recent request
  with non-zero `tokens_saved`
- Hermes chat test on a long prompt completes without quota error
  (or with reduced consumption vs baseline)
- Provider base_url in Hermes runtime points to `127.0.0.1:8788`,
  not the official host

If any of these fail, **the route is not working**, even if the system
"looks healthy" from outside.

### Failure Modes

| Symptom | Likely cause | What to investigate |
|---------|--------------|---------------------|
| `401 Unauthorized` | Headroom not passing Authorization header | Headroom version ≥ 0.26; `is_chatgpt_auth` branch triggered |
| Direct connection to upstream despite plugin enabled | plugin not loaded or auth.json base_url not rewritten | `config.yaml` plugins.enabled contains `headroom-route` |
| Headroom 502 Bad Gateway | Upstream OAuth endpoint URL changed | Update `route.yaml` anthropic_api_url |
| Kompress very slow | CPU backend or max_concurrent too low | Set `HEADROOM_KOMPRESS_BACKEND=pytorch` and provide GPU |

### Performance Baseline

GTX 1060 6GB, `max_concurrent=1`, `protect_recent=5`:

| Scenario | Original | Compressed | Saved | % |
|----------|---------:|-----------:|------:|--:|
| Short prompt (<500 tokens) | 458 | 458 | 0 | 0% |
| Long conversation (heavy tool results) | 28,368 | 11,283 | 17,085 | 60.2% |
| Many tool schemas loaded | 30,007 | 28,496 | 1,511 | 5.0% |

Key takeaways:

- Short prompts are skipped by design (`min_tokens_to_crush=500` threshold)
- Tool-result-heavy scenarios see the largest gains
- Kompress is stateless streaming compression — per-request latency ~150ms

### What I Recommend You Explore

This article intentionally avoids step-by-step commands. An AI agent
should be able to figure out the right commands from the constraints
above. Concretely, explore:

- How does Headroom discover its configuration? (env vars + config file)
- How does the plugin decide which providers to patch? (config.yaml block)
- How is OAuth token persisted across restarts? (auth.json)
- How can I tell after the fact whether traffic went through Headroom? (logs)
- What is `is_chatgpt_auth` and why does it matter? (branch in upstream detection)

### Compatibility Notes

Hermes plugins are independently loaded. Principles:

- Plugin names must be unique
- Do not mutate global state in `register()` unless intentional
- Do not contend over the same state file

The `headroom-route` plugin coexists with `quota-gate` (quota check
before LLM call) — no overlap, both run.

### Rollback

- Disable plugin: remove `headroom-route` from `config.yaml` plugins.enabled
- Restore `auth.json.credential_pool.*.base_url` to official URLs
- Restore `HERMES_OVERLAYS.providers.*.base_url` to official URLs
- Stop Headroom proxy (optional)

---

## When To Use This vs. Alternatives

| Approach | When to prefer |
|----------|----------------|
| **Headroom + Kompress (this article)** | You control the host, have a GPU, want zero API changes, want OAuth preserved |
| Hosted compression SaaS | You cannot install local dependencies; willing to send context to a third party |
| Smaller context window / shorter prompts | You don't actually have a compression problem |
| Switch to a different model | Sometimes the right answer is "use a cheaper model" not "compress more" |

---

## Closing

A working Headroom + Hermes setup gives you **transparent context compression**
without rewriting Hermes or swapping API keys. The hard part is OAuth
provider base_url rewriting — once solved by the plugin, the rest is
standard reverse-proxy deployment.

If you are an AI agent reading this and trying to set it up: focus on
**getting the verification checklist to pass**, not on copying any
specific command sequence. The constraints and invariants tell you
what success looks like; the means are deliberately left for you
to figure out.

---

## 📡 AI Agent Index (llms.txt fragment)

This article is part of the **AI Agent Tutorials** collection.
Other entries:

- (Coming soon) TradingView × Hermes — BTC divergence monitoring
- (Coming soon) X Content Pipeline — multi-platform publishing from Obsidian

Full index at the canonical blog's `llms.txt`.

---

📱 **More agent tutorials**: [link to author site]
🔗 **Canonical**: this article's canonical version lives at the author's blog.
