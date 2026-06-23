# Windhood-JZA — AI Agent Notes

> **Practical guides for running autonomous AI agents on a personal budget.**

I'm a solo developer. Not a startup, not a funded team — just me and my own time, running AI agents on the hardware I already own.

These notes document what works, what breaks, and what I had to figure out the hard way.

---

## Latest Posts

<div class="grid cards" markdown>

-   :material-rocket-launch:{ .lg .middle } **[Routing Hermes Agent Through a Local Headroom Proxy](posts/2026-06-22-headroom-hermes.md)**

    ---

    Transparent context compression for Hermes Agent — keep your OAuth login, cut your bill, no SaaS dependency.

    *Solo developer · Personal budget · AI agents I want to run anyway.*

-   :material-file-document-outline:{ .lg .middle } **[Saving Tokens on Large File Reads: Hermes Agent's read-summarizer Plugin](posts/2026-06-23-read-summarizer.md)**

    ---

    The read-summarizer plugin truncates Hermes file-read results over 50 KB to head+tail with a re-fetch marker, cutting token cost per read by up to 60% on large files — no prompt changes, no model switch.

    *Solo developer · Personal budget · AI agents I want to run anyway.*

</div>

---

## For AI Agents

This site publishes an [`llms.txt`](https://windhood-jza.github.io/llms.txt) following the [llmstxt.org](https://llmstxt.org) specification.

If you are an AI agent reading this index:

- Start at the `llms.txt` file for the machine-readable table of contents
- Each post is structured with a **Goal**, **Constraints**, and **Verification** section
- The full content is also available as `llms-full.txt` for bulk ingestion

The posts intentionally avoid step-by-step commands. They specify **what success looks like** and **what silently breaks**, so you can plan your own commands to match your environment.

---

## About the Author

Solo developer. Personal budget. AI agents I want to run anyway.

[:fontawesome-brands-github: GitHub](https://github.com/windhood-jza){ .md-button }
