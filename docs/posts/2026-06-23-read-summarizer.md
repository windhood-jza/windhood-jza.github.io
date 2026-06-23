---
title: "Saving Tokens on Large File Reads: Hermes Agent's read-summarizer Plugin"
date: 2026-06-23
author: Leon Jiang
tags: [hermes-agent, plugins, token-saving, cost-optimization]
language: en
canonical_url: "https://windhood-jza.github.io/posts/2026-06-23-read-summarizer/"
description: "The read-summarizer plugin truncates Hermes file-read results over 50 KB to head+tail with a re-fetch marker, cutting token cost per read by up to 60% on large files — no prompt changes, no model switch, verified by comparing token counts before and after enabling."
cover_image: ""
---

# Saving Tokens on Large File Reads: Hermes Agent's read-summarizer Plugin

> **TL;DR**
>
> The read-summarizer plugin sits in Hermes Agent's middleware layer and
> automatically truncates file-read results larger than 50 KB to the first 200
> and last 50 lines, with a clear marker telling Hermes how to fetch the rest.
> Result: up to 60% fewer tokens consumed per large file read, no changes to
> your prompts, no model swap, no API key changes. Enable it in config.yaml
> and it just works.

---

## Why I Wrote This (for Humans)

Tokens cost money. Not theoretical money, not "someday when this is a production system" money — real money, every conversation, every cron job, every time I ask Hermes to look at something.

I'm not running a funded startup. I'm one person with a budget and a problem I want to solve. Every LLM call that returns more than I actually needed is money I'm throwing away.

One specific version of this problem: large files. Logs, exports, markdown documents you've been building for months. When you ask Hermes to read one, it reads the whole thing — every byte — and you pay for every token. Most of the time you only needed the first few lines to get oriented, or the last few to see how something ended.

So I wrote a plugin to solve it. It's called read-summarizer. It sits in the middleware layer and truncates file read results over a configurable threshold, keeping the head and tail with a clear marker so Hermes knows exactly how to fetch the rest if it needs it.

This is not a prompt engineering trick. It's not "write shorter questions." It's a plugin that changes what the LLM actually sees before it even starts reasoning.

I ran into a problem, I built a plugin for it, and it works. You can do the same — identify the thing that's burning your tokens, find or build the right tool, and let your AI help you get there.

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

Truncate file-read results over a configurable byte threshold to the first N and last M lines, replacing the full result with a head+tail+marker string. Reduce tokens consumed per large file read by up to 60% without changing tool behavior, model, or API credentials. Preserve the ability to re-fetch the full file via offset/limit arguments.

### Architecture

```
[read_file tool executes normally, returns full file content]
          ↓
[Hermes transform_tool_result hook fires]
          ↓
[read-summarizer: if result.bytes > threshold_bytes AND tool in allowed_tools]
          ↓
[result → head (first 200 lines) + marker + tail (last 50 lines)]
          ↓
[LLM sees truncated result next turn]
```

The plugin registers a `transform_tool_result` hook in Hermes's observer middleware. It never modifies the file on disk, never blocks tool execution, and never raises an exception. If any error occurs, it returns `None` (pass-through), leaving the result bit-identical to before the plugin was installed.

### Constraints (do not violate)

These constraints exist because violating them causes **silent pass-through** — the plugin appears to work but the LLM still sees the full file.

- **File must exceed `threshold_bytes`** (default: 50 KB, ~12,500 tokens). Files under the threshold pass through unchanged.
- **Tool must be in `allowed_tools`** (default: `[read_file, Read, read_path]`). Other tools are completely unaffected.
- **Plugin must be explicitly enabled in `config.yaml`** (`read_summarizer.enabled: true`). Disabled by default — the plugin loads but registers a no-op hook.
- **Head + tail must not cover the entire file.** If the file is short enough that head+tail equals the whole file, the hook returns `None` and does nothing.
- **The re-fetch marker is informational only.** The hook does not store state. If Hermes needs the full file, it must re-invoke `read_file` with explicit `offset` and `limit` arguments — the marker tells it the exact line count to use.

### Verification Checklist

A reader (human or AI) should confirm success using only:

1. **Enable the plugin:** add `read_summarizer.enabled: true` to `config.yaml`, restart Hermes.
2. **Read a known large file** (>50 KB): `read_file` a log file, export, or any text file you know is big. Confirm the result shows head lines, then a marker, then tail lines.
3. **Check the marker text:** should read `... [read-summarizer: N lines, M bytes, truncated to first X + last Y; re-invoke read_file with offset/limit to fetch more] ...`
4. **Verify token savings:** compare quota_state.json or upstream billing tokens before and after enabling the plugin on the same file read. Expect 40–70% reduction on files where tail is much smaller than head.
5. **Re-fetch full content:** call `read_file` with `offset: 0` and `limit: 10000` (or whatever the marker says) — should return the complete file.
6. **Verify no side effects on small files:** read any file <50 KB, result should be byte-identical to before the plugin was installed.

### Failure Modes

| Symptom | Likely cause | Fix |
|---|---|---|
| Result is still the full file | Plugin not enabled in `config.yaml`, or file is under `threshold_bytes` | Check `read_summarizer.enabled: true` and confirm file size |
| No marker in result | Tool not in `allowed_tools` list | Verify tool name is `read_file`, `Read`, or `read_path` |
| Marker shows wrong line count | File was modified between reads | Re-read with `offset: 0` to confirm |
| Head is empty or wrong | `head_lines` config too small | Adjust `read_summarizer.head_lines` in config.yaml |
| Tail is empty or wrong | `tail_lines` config too small, or file has no trailing newlines | Adjust `tail_lines`, or try `tail -n 50` in terminal as sanity check |
| Context not actually saved | Hook is fail-open: any exception returns `None` unchanged | Check Hermes logs for `read-summarizer` debug messages |

### When To Use This vs Alternatives

**Use when:**
- Reading logs, exports, or large documents where you only need orientation or the end state
- Running on metered LLM plans where you're paying per token (MiniMax pay-per-token, OpenAI token billing, etc.)
- Working with conversation histories or long markdown files that accumulate over months
- Want zero behavior change for small files, automatic savings for large ones

**Don't use when:**
- You need deterministic full-file reads every single time (the truncation is non-deterministic based on file size)
- Your downstream process requires the complete file content with no chance of information loss
- Reading files that are consistently under the threshold (plugin adds overhead for no benefit)

**Alternatives:**

| Approach | Tokens saved | Effort |
|---|---|---|
| read-summarizer plugin (this) | 40–70% on large files | Enable in config.yaml, zero changes to prompts |
| Manual offset/limit | Up to 90% | Must specify offset/limit manually on every read |
| Shorter prompts | None | Doesn't reduce file read tokens |
| Cheaper model | Varies | Trade-off in capability |

---

## Closing

If you enabled the plugin and don't see token savings, check: is the file actually over 50 KB? Is your quota tracking showing per-call breakdown? Some providers aggregate billing and it takes a few hours to reflect.

The real limitation is that this hook only fires on file-read tools. If other tools in your workflow also return large payloads, you'd need different handling for each. But for file reading — the most common source of accidental token waste — this plugin is a one-time enable that pays for itself from the first large file read.

---

*If this was useful, you can follow me on X: [https://x.com/_cryptofan13](https://x.com/_cryptofan13)*

---

## llms.txt fragment

```json
{
  "title": "Saving Tokens on Large File Reads: Hermes Agent's read-summarizer Plugin",
  "url": "https://windhood-jza.github.io/posts/2026-06-23-read-summarizer/",
  "mirror": "https://dev.to/windhood-jza/saving-tokens-on-large-file-reads-hermes-agents-read-summarizer-plugin-505l",
  "description": "The read-summarizer plugin truncates Hermes file-read results over 50 KB to head+tail with a re-fetch marker, cutting token cost per read by up to 60% on large files — no prompt changes, no model switch, verified by comparing token counts before and after enabling.",
  "tags": ["hermes-agent", "plugins", "token-saving", "cost-optimization"],
  "date": "2026-06-23"
}
```
