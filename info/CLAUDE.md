# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A planning and research workspace for **Zuri** (zuri.com, about 30 people, drones). The goal is an AI system that chats over a curated set of Google Drive documents per project. There is **no code yet**, so there are no build, lint or test commands. Everything here is Markdown. Since 2026-10-29 these files live in the `info/` folder of the git repository `https://github.com/illich/mozek.git` (local clone: `/run/media/mike/encrypted-usb/vibe/mozek`); the old copy in `/run/media/mike/encrypted-usb/vibe/ai4zuri` is frozen and must not be edited. Commit each round of edits with a short message.

## Document hierarchy (read in this order)

1. `concept.md`: the original requirements, **written by the user**. Don't edit it unless asked.
2. `master-plan.md`: the three-phase plan, **written by the user**. It's authoritative and the other documents must follow it. Don't edit it unless asked.
   - Phase 1: context service + minimal admin web UI; chat happens in Claude.ai and in Slack via Claude Tag.
   - Phase 2: our own chat app ("Zuri Hub"), Zuri leaves Slack, single tenant.
   - Phase 3: public SaaS at about $5/user/month.
   - Identity note: everyone at Zuri has a Google account and a Zoho Mail account, both with OAuth, so there is no password store of our own.
   - `design.md` (also **written by the user**, 2026-10-29): the Phase 2+ UI layout. Icon rail (Home, DMs to be decided, projects sorted by the user's usage with a "…" overflow after five, Settings, User), a top-bar search, a narrow conversation list with a per-project Files item, and a wide main pane (conversation, or the Files list with maintainer and sync status). Don't edit it unless asked; `zuri-context-service.md` §6.3 follows it.
3. `zuri-context-service.md`: the detailed design. §1–§5 cover Phase 1 (steps 1.0–1.3), §6 covers Phase 2 (steps 2.0–2.4), §7 covers Phase 3 (steps 3.1–3.4), and §8 lists open questions per phase.
4. `solutions.md`: the market and option survey (requirements R1–R14, options A–I, comparison table). Option I is the chosen architecture, and it points to `zuri-context-service.md` for details.
5. `existing-solutions-context-service.md`: the build-vs-buy research report (verdict: build, reusing FastMCP 4.x, the Google Picker sample, Drive Markdown export, the Docs API suggestion mode and the "LLM Wiki" pattern). An identical copy is kept at `reports/Existing context service solutions.md`. **Update both together.** The raw research notes are in `research_notes/`: round 1 in `Existing context service solutions/`, round 2 (2026-10-29, re-verification of every platform claim plus 18 more vendors) in `Round 2 (2026-10-29)/`. Notes are append-only; add a new round folder rather than rewriting old notes.

When `master-plan.md` or `concept.md` changes, propagate the change through 3 → 4 → 5, and keep the cross-references (section numbers, step numbers) consistent.

## Key decisions that shape all documents

- **Billing constraint:** Claude subscription logins may only be used in Claude.ai and Claude Code, not in third-party or self-built apps (including the Agent SDK, or `claude -p` running as a server backend). Subscription-billed chat therefore has to go through Claude.ai with our remote MCP server as a custom connector. Anything in our own app, and background jobs such as the summarizer, is API-billed. Since 2026 the Team plan includes pooled API credits ($20 per Standard seat, capped at $500/month per org) that pay for that API usage first.
- **Claude Tag identity:** a connector attached to a Slack channel runs under one shared agent identity (an admin signs in once). Per-user roles in channels only work through personal connectors (on by default on Team) or in DMs. Claude Tag is public beta, Team/Enterprise only, and its launch credit expired on 1 October 2026.
- **MCP auth stack:** FastMCP 4.x with `GoogleProvider` (OAuth proxy). The MCP spec revision is 2026-07-28, which deprecates Dynamic Client Registration in favour of Client ID Metadata Documents. Enforce the Zuri domain on the `email` claim server-side; there is no `hd` claim. Google is the only login in Phase 1 (the Picker needs it, and one FastMCP server has one interactive login provider). Zoho Accounts is an OpenID Connect provider and becomes the second login for the Phase 2 Hub; the registry stores `(provider, sub)` per user and links by verified email.
- **Core architecture:**
  - registry (projects ↔ documents many-to-many, one maintainer per doc, roles admin/member/reader);
  - Drive sync worker → Markdown mirror in git (diffs come from git history); `documents.status` uses the fixed vocabulary `not_fetched | fresh | outdated | error`, shared by the Phase 1 admin page and the Hub's Files view;
  - incremental per-project `SUMMARY.md` plus a "lint" contradiction pass → issues sent to maintainers;
  - FastMCP remote MCP server with Google OAuth and per-user role checks.
- **Drive access:** only the `drive.file` scope (Picker) or a service account the picked files are shared with. Never restricted scopes: they're unnecessary, and in Phase 3 they would trigger Google's CASA security assessment.
- **Phase 3 readiness rules** (zuri-context-service.md §7.4): `tenant_id` everywhere, per-tenant git paths, no Zuri logic in MCP tools, tool annotations from day one.
- **Phase 2 chat app is our own build,** not an Open WebUI or Mattermost fork, because of licence and branding limits on reselling in Phase 3.

## Writing conventions used in these documents

- Mark claims from third-party-only sources **(unverified)**. Mark inferences that still need testing **(verify)**.
- Durations and costs are rough estimates and are labelled as such.
- Each research-based document ends with a Sources list of URLs. Keep it when editing. `zuri-context-service.md` has "Sources added in revision 4" and "Sources added in revision 5" sections for the same purpose.
- Revisions are numbered in each document's status line, and the design doc keeps a short "What changed in revision N" list under it.
- Today's research context is October 2026 (last full re-verification: 2026-10-29). Vendor features, especially betas like Claude Tag, change fast, so re-check before relying on them. Known open items to re-check: claude-ai-mcp issue #476 (connector tools not surfaced on claude.ai web), whether the Markdown export includes all Docs tabs, whether connectors work inside shared Claude.ai projects, and whether everyone's Google and Zoho Mail addresses are identical.
