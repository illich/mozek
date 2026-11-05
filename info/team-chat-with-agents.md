# Team chat products where AI agents are members: inspiration for the Hub

Status: research note, 2026-11-05 (revision 1). Starting point: a hand-made list of twelve "Slack-style chat + agents" products. Each was checked against its own site, docs, repo and pricing page on 2026-11-05. Only the products that fit the frame of our plan are listed here. Claims that rest on a single blog post or tweet are marked **(unverified)**. These products move fast (one of them shipped 117 releases in the six days before this note, three repos were pushed on the day of writing), so re-check before relying on any detail.

**Fit test used.** A product is in if (a) channels or rooms plus human DMs or groups are the daily surface, as in [design.md](design.md), and (b) the AI is a *participant in the conversation* that reads and answers in rooms, rather than mainly an executor of coding tasks or a "hired workforce". Products that are developer pipelines (agents posting pull requests, Claude Code / Codex hand-off rooms) or agent platforms without human-to-human chat are left out.

**How to read this.** None of these products does what our plan is built on: hand-picked external documents as the AI's context and one human maintainer per document. They are inspiration for the Hub's chat layer (Phase 2) and, for the hosted ones, the competitive set for Phase 3. The cross-cutting observations are in §3.

---

## 1. Summary

| Product | Maker, status | Licence, self-host | Price | AI in rooms | Shared context | AI-free rooms | MCP server | Relation to our plan |
|---|---|---|---|---|---|---|---|---|
| **Ando** | Asari Inc., SF; out of stealth 24 Sep 2026, waitlist; $20M from Accel, Index, Emergence | Closed SaaS, no self-host | Per human seat, not published | Yes, agents are members with inbox and permissions | In-app wiki (Dictionary, channel context, journal); Notion; no Drive | **Yes, explicit guarantee** | Yes, OAuth, "Connect as me" | Closest product to the Hub; Phase 3 competitor; whisper and wiki patterns |
| **agentes.chat** | Codari Global LLC (Wyoming; pt-BR first language **(unverified)**); early access since 29 Oct 2026 | Says AGPL self-host, but **no public code, image or npm package exists yet** | Flat per workspace: $29 / $49 (10 people) / $99 (25 people), ≈ $4–5 per user | Yes, router picks who answers | Wiki agents read; MCP connectors; no Drive | Partial (membership) | Yes, OAuth, for Claude.ai | Phase 3 competitor at our price point; approval cards, responsible person per agent |
| **Kluff** | Kluff, Inc. (Delaware); shipped, all platforms | Closed SaaS | Contradictory: $15/user/month (pricing page) vs "not per seat" (docs) vs Free/Team/Org (home) | Yes, `@kluff` on mention; agent threads in projects | Per-project files and memory; Drive as agent tool | Not documented | No | Phase 3 competitor; per-room AI scoping model |
| **Inline** | Mo Rajabi and Dena Sohrabi; macOS/iOS beta, web planned | AGPL-3.0 core, Apache-2.0 SDKs; self-host "planned" | Not published | Yes, bridged coding agents as bots in threads | Threads only; no document library | Partial (follow semantics) | Yes, OAuth, scoped to chosen spaces | Thread-first UX; scoped MCP consent screen; Apple-only |
| **Buzz** | Block, Inc.; launched 21 Jul 2026, "not finished", 36.7k stars | **Apache-2.0**, self-host (compose, Railway) | Free; hosted beta free | Yes, agents are members with own keys | Channel history, canvases; no Drive | Yes (membership, relay-enforced) | Partial (agent-side only) | §6.4 input: permissive codebase, but Nostr onboarding and dev-tool core |
| **Flow (Freeflow)** | Scott Persinger, SF, one maintainer; hosted, 6 stars | **FSL-1.1-ALv2** (source-available, competing-use clause, Apache after 2 years); self-host documented | Free, no seat limits | Yes, agents sponsored by a human member | Channel history; no Drive | Partial (private channels) | Partial (agent → chat direction) | §6.4 input: Slack-derived codebase with a licence that blocks a Phase 3 fork |
| **Cumora** | `yetone` (China **(unverified)**), no legal entity; invite-only preview, 4.1k stars | **MIT**, self-host (Node, Postgres, Redis) | Free during preview; tiers in code | Yes, agents on the roster with personas | Per-agent memory; no Drive | Partial (membership) | No | Codebase reference only: no channels or threads, agent-team positioning |

---

## 2. Products

### 2.1 Ando — https://www.ando.so

**What it is.** A "Slack alternative for humans and AI agents": public and private channels, DMs, group DMs, threads, search, live audio rooms with transcripts, a wiki, and a Slack import/sync bridge. Agents are workspace members with their own identity, inbox and permissions. They can be Ando-hosted (pick a model from OpenAI, Anthropic, Google, xAI) or brought in from outside (Claude, Codex, Hermes, OpenClaw, Devin, custom runtimes).

**Status.** Out of stealth on 24 Sep 2026; waitlist-gated, app live; SOC 2 Type II; customers in 15 countries (press). Desktop Mac/Windows and iPhone/iPad/Android; Linux is listed on the FAQ but not in the docs. No public code.

**Pricing.** Per human seat, rates "shared during onboarding"; agent participation is not metered ("a philosophy, not a promise of unlimited usage").

**What matches our concepts.**
- *Visibility contract:* "Agents see a DM or private channel only when they are in it"; participant lists are always visible; sharing DM content into an agent's view requires explicitly forwarding selected messages; the optional nightly "dreaming" that compiles discussions into wiki pages reads only public channels. This is the whisper guarantee stated as product policy.
- *Wiki as shared context:* workspace pages, per-channel context plus a daily journal, member context, and a **Dictionary** of facts and terms, each with a source and a "retire" action; page versions carry the author; people edit, agents read and update via MCP. Import is Markdown only; Google Drive is not mentioned.
- *Roles:* owners/admins, full members vs guests; per-agent access level (Full / Read and reply / Read only), per-agent tool allowlist, per-channel proactivity level (Mentions / Low / Medium / High) and notification setting.
- *MCP server:* hosted Streamable HTTP endpoint with OAuth for interactive clients and API keys for headless ones; tools over members, conversations, messages, wiki, transcripts and files; "Connect as me" from Claude desktop/web and ChatGPT.

**Borrow.** The visibility contract, the Dictionary (facts with a stated source, retirable, versioned with author), and the "connect as me vs add as agent" choice at MCP pairing.

**Relation to our plan.** The closest shipped product to the Phase 2 Hub and a direct Phase 3 competitor, well funded. It has no curated external document set and no per-document maintainer; its wiki is written by people and agents inside Ando. Pricing unknown.

### 2.2 agentes.chat — https://agentes.chat

**What it is.** A Slack/Teams-style workspace (public and private channels, threads, DMs, tasks, wiki, Drive, calendar, routines) where agents are teammates: each has a name, a role template, a model, tools with access levels, memory, a monthly budget and a **responsible person**. Sensitive actions become an approval card in the chat that shows the exact action and expires after 24 hours. "Built for any team, not just engineers."

**Status.** Early access with invites in waves; intro post dated 29 Oct 2026; 117 releases between 30 Oct and 5 Nov 2026. Apps for browser, desktop (Mac/Windows/Linux) and phones per the vendor; store listings not independently verified. No independent reviews found.

**Licence and self-host.** The site states an AGPL self-hosted edition with an installer script and Docker Compose. As of 2026-11-05 **nothing is publicly available**: the GitHub org has no public repos, the repo the installer points to returns 404, the container image denies anonymous pulls, and the installer's npm package returns 404. Treat the open-source offer as a plan.

**Pricing.** Flat per workspace, AI always on the customer's own keys or plans: Free (self-hosted), Personal $29/month, Team $49/month for up to 10 people (+$4 per extra person to 15), Business $99/month for up to 25 (+$4 to 50). At full seats that is about $4–5 per user per month. The About page and intro post, dated the same week, say pricing "hasn't been announced".

**What matches our concepts.**
- *Routing:* a small router model (or code rules) picks one agent per unmentioned message, with caps on chain length; every message shows who picked the agent; an agent can pass silently.
- *Context:* agents read the company wiki when answering and write to it only when asked; wiki has links, backlinks and an Obsidian vault import; per-agent memory where agent-proposed changes to its own instructions need human acceptance, with history. Read-only Postgres connections and 58 MCP connector presets. No Google Drive connector documented.
- *Roles and limits:* only owners/admins create agents; per-agent tool access none/read/read+write; per-agent reachability (everyone / chosen people / owner only); per-person usage limits; audit log; approvals go to the agent's responsible person or admins.
- *MCP server:* the workspace is an OAuth MCP server (tools to send to an agent, wait for a reply, list agents and channels, read a conversation); documented for Claude.ai, Claude Desktop, Claude Code and others; acts as the signed-in user and never approves actions.

**Borrow.** The approval card with the exact action and expiry, the "who picked this agent" line under answers, the responsible-person-per-agent idea (the nearest analogue to our maintainer-per-document), and agent-proposed edits that need human acceptance.

**Relation to our plan.** The only product found at roughly our Phase 3 price point, and it names Claude.ai as an MCP client. It is a week old, invite-only, and its open-source claim is unbacked today. Watch it.

### 2.3 Kluff — https://kluff.com

**What it is.** A hosted "collaborative AI workspace" built around team chat: Topics (shared threads with an explicit audience), DMs (1:1 and small group), Projects (people, a main channel, agent threads, files) and audio/video "Jams" with captions and AI summaries. `@kluff` can be mentioned in any topic, DM or project channel; inside projects, dedicated agent threads treat every message as directed at the agent. Marketed to Sales, Marketing, Ops, IT, Finance, CX and People teams, not developers.

**Status.** Shipped; iOS 1.1.2 (7 Oct 2026), Android, macOS, Windows, Linux, web and a Chrome side panel. Accounts are created by a workspace admin only. No founders, team or location published anywhere; the GitHub org is empty.

**Licence.** Proprietary SaaS, no self-host.

**Pricing.** Internally inconsistent: the pricing page says Business $15/user/month (three-user minimum, 14-day trial, "unlimited AI usage" under fair use) plus a custom Enterprise tier; the settings docs say "Kluff does not charge per seat"; the home page lists Free, Team and Org plans.

**What matches our concepts.**
- *AI scoping per room:* in topics and DMs the agent acts only on mention and "does not silently publish the conversation"; projects add agent threads, a per-project allowlist of integrations the agents may use, and an "ask first / automatic" mode for tool use.
- *Context:* per-project uploaded files that agents read when needed, per-project memory ("so a sales workspace doesn't leak into an infra one"), git-backed projects. Google Drive, Docs and Sheets are agent *tools* (search, read, manage), not a curated document set. Company memory is described as automatic capture, not curation.
- *Models:* OpenAI, Anthropic and Google routed through Kluff; only "faster / default / thinking" tiers are exposed; no local models, no own key. PII redaction before any model call.
- *MCP:* client yes (organisation-level MCP servers with scopes); no MCP server.

**Borrow.** The "human-first vs agent-first" split: mention-only AI in ordinary rooms, dedicated agent threads plus a tool allowlist and approval mode per project.

**Relation to our plan.** A shipped Phase 3 competitor aimed at the same non-developer audience. Its context model is files uploaded into Kluff plus automatic memory, the opposite of "Drive stays the master". Nothing says unmentioned room content is excluded from the company memory, so it has no whisper equivalent.

### 2.4 Inline — https://inline.chat

**What it is.** "A thread-based chat app for all work, with your teammates and agents." Native macOS and iOS client with Spaces, numbered and linkable threads, reply threads, DMs, user groups, voice rooms and search. Agents join as bots through a local "agent bridge" that runs Codex, Claude Code, OpenCode, Amp, Hermes or OpenClaw on a member's machine; an MCP server, CLI, Bot API and SDKs let external assistants read and post. Grew out of the makers' earlier chat app Noor. The X post in our source list is a personal usage note from 30 Oct 2026, not a launch announcement.

**Status.** Beta: macOS 15.2+ and iOS 18+ (TestFlight), "web coming soon", no Windows, Android or Linux. Repo `inline-chat/inline`: 908 stars, 6,392 commits, pushed 2026-11-05. Legal entity and location not stated (Estonia per the founder's LinkedIn **(unverified)**).

**Licence and self-host.** Apps and server are **AGPL-3.0-only**; SDKs, CLI, MCP package and protocol schema are Apache-2.0. Self-hosting is "planned", not offered. No pricing published; the changelog mentions "Pro spaces" without a price **(unverified what Pro includes)**.

**What matches our concepts.**
- *AI as participant:* "a thread's first message decides which bots follow"; `/follow` and `/unfollow`; mention, reply and DM wake rules; one native agent session per thread; approvals shown in the conversation; up to 30 bots per account.
- *Visibility:* an agent sees only the threads it follows or is mentioned in and the chats it is added to. No explicit no-AI flag.
- *MCP server:* remote Streamable HTTP with OAuth; the consent screen lets the user **choose which spaces and conversation types** the external AI may read. Claude.ai is not named, but it is a generic remote MCP server.
- *Context:* the threads themselves (agents get a full thread as Markdown); thread backlinks are "an early step toward a knowledge graph". No document library, no Drive.

**Borrow.** The scoped MCP consent screen, and the thread-follow semantics for bots.

**Relation to our plan.** Thread-first rather than channel-first, Apple-only, AGPL, no self-host, no pricing, and the agents are bridged coding harnesses. Good UX reference for lightweight threads and for a scoped MCP server; not a base to build on.

### 2.5 Buzz by Block — https://github.com/block/buzz

**What it is.** A self-hostable Nostr relay (Rust) plus a Tauri/React desktop app where every message, reaction, workflow step, approval and git event is a signed event in one log. Channels (open or private, Stream or Forum), threads, DMs up to nine people, canvases (a shared document per channel), media with pinned comments, voice huddles, YAML workflows and an experimental Git forge. Agents are first-class members with their own keypairs. The README calls it "another AI-adjacent developer tool".

**Status.** Launched 21 Jul 2026; 36,744 stars, 4,885 forks, pushed 2026-11-05; README: "Not finished. We will tell you what works and what doesn't." Desktop for macOS, Linux and Windows; a Flutter mobile client is in development; push notifications planned. Hosted communities are free during beta with no published limits (3 communities per account, 180-day retention, 5 GB, hosted messages readable by Block for moderation, all **(unverified)**, single secondary source).

**Licence and self-host.** **Apache-2.0.** Docker Compose (Postgres, Redis, MinIO, optional Caddy), a Railway template, or from source (Rust, Node, pnpm).

**What matches our concepts.**
- *Roles:* member roles Owner, Admin, Member, Guest, Bot; channel membership is "the only gate", enforced by the relay; guests are scoped to specific channels.
- *AI-free rooms:* private channels are hidden and invite-only; fan-out explicitly excludes non-members; an agent not added to a channel cannot see it. Structurally the closest match to whisper rooms among the open codebases.
- *Models:* model-agnostic; an ACP harness for goose, Codex and Claude Code; the `buzz-agent` crate speaks the Anthropic Messages API, OpenRouter and any OpenAI-compatible endpoint including Ollama; "Buzz Mesh" pools members' GPUs as a local provider.
- *Context:* permission-aware full-text search over channel history, canvases and repos; "living documents, collaboratively editable by humans and agents via MCP tools" in the vision doc. No Drive, no maintainer.
- *MCP:* client yes (per-agent MCP servers); the MCP tool surface is consumed by attached agents, and no remote endpoint for external clients such as Claude.ai is documented.

**Borrow.** Identity-scoped agents (the agent is a member with its own key and memberships, so "what the AI can see" is just membership), the five-role model, and the hash-chained audit log of every agent action.

**Relation to our plan.** The only large, permissively licensed codebase with the full chat surface. Against it: onboarding requires generating and backing up a Nostr keypair, agents are attached by installing a coding harness on a machine, mobile is unfinished, and the Git forge and patch review sit at the core. Relevant to the build-vs-fork question in zuri-context-service.md §6.4 as a source of patterns and components, less as a base.

### 2.6 Flow (Freeflow) — https://freeflow.im

**What it is.** A Slack-derived team chat (workspaces, channels, threads, DMs, group DMs, huddles, files, reactions) where coding agents join as full members through a bridge daemon run by a sponsoring human member (`npx flow-agent-bridge <invite code>`, which spawns Claude Code, Codex or "any prompt in, text out CLI"). One of four tools under the Freeflow umbrella; the repo is adding Mail and Calendar tabs. The README says Flow "has been constructed 100% via coding agents (mostly Claude)".

**Status.** Hosted and free at app.freeflow.im; repo `freeflow-im/flow` created July 2026, daily commits, 6 stars, 0 releases; native macOS app, iOS app on the App Store (publisher "BizTrip AI"), Electron for Windows/Linux from source. Effectively one developer (Scott Persinger, 669 of about 689 commits).

**Licence and self-host.** The site says "free + open source", but the licence is **FSL-1.1-ALv2**: source-available, forbids "competing use" (offering the software, or substantially similar functionality, in a commercial product), and converts each version to Apache-2.0 two years after release. The agent bridge package is MIT. Self-hosting is documented: one Node process, Postgres 16, NATS, blobs on disk or S3.

**What matches our concepts.**
- *AI as participant:* presence, DMs, mentions, threads, attachments, "thinking…" progress; agents can create channels and join huddles; bridge config has `respondToAgents`, `eventScope` and `allowedTools`.
- *Roles:* workspace owner / admin / member; private channels, but any member can add an agent; no "AI cannot see this room" concept.
- *Models:* the chat server holds no model keys at all; the model is whatever harness the sponsoring human runs, so local models work if the CLI supports them.
- *Context:* channel and thread history only; no Drive; the separate MCP Docs product edits files over MCP.

**Borrow.** The sponsorship model (every agent is traceable to one accountable human member, onboarded with an invite code) and agent artefacts shown in a side panel behind a token-checked proxy.

**Relation to our plan.** The most Slack-shaped open codebase, but the FSL competing-use clause rules out forking it for Phase 3 until each version's Apache conversion, and the bus factor is one. Pattern reference for §6.4, not a base.

### 2.7 Cumora — https://cumora.ai

**What it is.** Open-source cross-platform team chat (Electron desktop, PWA, iOS beta) "where AI agents are first-class participants alongside humans: same roster, same DMs, same group conversations, same Kanban board and calendar." Agents have personas and private memory, can act unprompted on a timer, claim Kanban cards and send email. Two brain paths: Cumora Cloud (per-agent Kubernetes pod on OpenAI) or bring-your-own-agent (a daemon on your Mac or VPS driving a local Claude Code or Codex). Tagline: "Where agent teams gather."

**Status.** Invite-only preview, free; repo `yetone/cumora` created 17 Aug 2026, 4,128 stars, pushed 2026-11-01; desktop v0.18.9 (29 Oct 2026) for macOS, Windows and Linux; iOS on TestFlight; Android source only. Author is the maintainer of avante.nvim; no company or legal entity named.

**Licence and self-host.** **MIT.** Node/Express plus WebSocket server, Postgres, Redis; an OpenAI key is the only hard requirement (for cloud agents), everything else soft-disables. Plan tiers free / pro / max exist in code, no prices.

**What matches our concepts.**
- *Chat surface:* roster, DMs, group conversations, Kanban, calendar. **Channels and threads are not documented.**
- *Visibility:* conversations have chosen members, so an agent-free group is possible; no "agents can never read this" statement. Note the inverted vocabulary: Cumora's "Whisper rooms" are humans peeking at agent-to-agent DMs, the opposite of ours.
- *Models:* cloud agents are OpenAI only; bring-your-own covers Claude Code, Codex and eight more CLIs; per-agent provider profiles allow a local Claude-compatible gateway on loopback.
- *Context:* per-agent memory, notes and workspace files; no external document source. MCP: agents get one fixed Cumora server with a single `cli` tool; no MCP server for external clients.

**Borrow.** The coordination design (a freshness check before posting so two agents never post stale duplicates, atomic Kanban claims, a cheap "small-brain" triage before waking the big model, a per-machine concurrency cap) and provider credentials kept on the paired computer rather than the server.

**Relation to our plan.** Included as a **codebase reference only**: an MIT-licensed human-plus-agent chat with working desktop, PWA and iOS clients is relevant to §6.4. As a product it is agent-team-centric, lacks channels and threads, and is invite-only.

---

## 3. What this means for our plan

1. **Nobody curates external documents.** None of the seven uses hand-picked Google Drive (or any external) documents as the AI's context, and none has a human maintainer per document. All of them keep context inside the app: a wiki (Ando, agentes.chat), uploaded project files and automatic memory (Kluff), or conversation history (Inline, Buzz, Flow, Cumora). The nearest analogues to our maintainer loop are Ando's Dictionary (facts with a stated source that can be retired) and agentes.chat's responsible person per agent. This confirms the differentiation argued in [existing-solutions-context-service.md](existing-solutions-context-service.md) from the chat side as well.
2. **AI visibility by membership is the common pattern.** Every product scopes what an agent sees by which rooms or threads it is a member of. Only Ando states it as an explicit guarantee ("agents see a DM or private channel only when they are in it", with forwarding as the sole way to share). Our whisper design (zuri-context-service.md §6.2, `ai_enabled = false` enforced in the backend) goes one step further by making it a property of the room rather than of membership. Worth keeping; worth stating as a product promise the way Ando does.
3. **The chat itself as an MCP server.** Ando ("Connect as me"), agentes.chat and Inline expose the workspace, its conversations and wiki to Claude.ai and other clients through an OAuth MCP server, with Inline letting the user pick which spaces the external AI may read. Our MCP server exposes documents and summaries only. Exposing Hub conversations the same way is an option for Phase 2 or 3, not a change to the current plan.
4. **Room-level AI etiquette.** Recurring, borrowable mechanics: mention-only AI in ordinary rooms with dedicated agent threads in projects (Kluff); a per-agent proactivity or response mode separate from tool permissions (Ando); "who picked this agent" shown under every answer and approval cards with an exact action and an expiry (agentes.chat); one accountable human per agent (Flow, agentes.chat). Our composer's Send / Ask AI split ([claude-design-handoff.md](claude-design-handoff.md) §4.2) is the same idea expressed in the UI.
5. **Licence input to §6.4 (build vs fork).** Buzz (Apache-2.0) and Cumora (MIT) are permissive; Flow (FSL, competing-use clause, Apache only after two years) and Inline (AGPL, no self-host yet) are not usable as a base for a resold product. None of the four has the data model we need (project-scoped rooms, `visible_from`, AI-off rooms, document registry), so the own-build recommendation stands. Buzz's role model and audit log, and Cumora's coordination rules, are the parts worth reading before building.
6. **Price points for §7.2.** agentes.chat sells at about $4–5 per user (flat per workspace, AI on the customer's keys), Kluff at $15 per user with AI included, Ando undisclosed, Buzz and Flow free. The "$5 for the chat and knowledge layer, AI paid separately" model already exists in the market in agentes.chat's form.

---

## Sources

- Ando: https://www.ando.so , https://www.ando.so/faq , https://www.ando.so/pricing , https://www.ando.so/features/ai-agents , https://www.ando.so/features/bring-your-own-agent , https://www.ando.so/features/wiki , https://www.ando.so/security , https://www.ando.so/blog/introducing-ando , https://docs.ando.so/ , https://docs.ando.so/docs/external-agents , https://cursor.com/marketplace/mcp/ando , https://tbreak.com/ando-agent-native-messaging/ ; press release (403 when fetched): https://www.globenewswire.com/news-release/2026/09/24/3368344/0/en/ando-launches-agent-native-messaging-platform-announces-20-million-seed.html
- agentes.chat: https://agentes.chat/ , https://agentes.chat/about , https://agentes.chat/news/introducing-agentes-chat , https://agentes.chat/pricing , https://agentes.chat/docs , https://agentes.chat/docs/apps-de-ia , https://agentes.chat/docs/install-on-a-server , https://agentes.chat/docs/wiki , https://agentes.chat/docs/memory , https://agentes.chat/docs/who-answers , https://agentes.chat/features/multi-agent-chat , https://agentes.chat/changelog , https://agentes.chat/terms.md , https://agentes.chat/install , https://github.com/agentes-chat (no public repos)
- Kluff: https://kluff.com , https://kluff.com/pricing , https://kluff.com/docs (topics, direct-messages, projects, agents, files-and-code, integrations, settings-and-billing, jams, apps) , https://kluff.com/integrations , https://kluff.com/security , https://kluff.com/terms , https://kluff.com/download , https://apps.apple.com/app/id6812050027 , https://github.com/KluffHQ
- Inline: https://inline.chat , https://inline.chat/docs , https://inline.chat/docs/agents , https://inline.chat/docs/mcp , https://inline.chat/docs/openclaw , https://inline.chat/docs/changelog , https://inline.chat/docs/security , https://inline.chat/docs/technical , https://inline.chat/download , https://inline.chat/legal/terms , https://github.com/inline-chat/inline (README, LICENSING.md, packages/mcp/README.md) , https://x.com/morajabi/status/2116144964564287503
- Buzz: https://github.com/block/buzz (README.md, VISION.md, VISION_AGENT.md, VISION_PROJECTS.md, ARCHITECTURE.md, AGENTS.md, docs/MCP_DRIVEN_HOOKS.md, crates/buzz-agent/README.md, mobile/README.md) , https://buzz.xyz , https://siliconangle.com/2026/07/21/block-launches-buzz-open-source-workspace-humans-ai-agents/ , https://runtimewire.com/article/block-buzz-projects-self-hosted-git-forge , https://www.dplooy.com/blog/buzz-by-block-setup-claude-code-agents-channels (hosted limits, unverified) , https://railway.com/deploy/buzz-relay-self-hosted-ai-agent-workspace--buzz-relay-agent-workspace
- Flow (Freeflow): https://freeflow.im , https://flow.freeflow.im , https://github.com/freeflow-im/flow (README, LICENSE.md, DEPLOYMENT.md, packages/agent-bridge/README.md, packages/shared/src/dto.ts, packages/server/src/db/schema.ts) , https://github.com/freeflow-im/flow/pull/654 , https://github.com/freeflow-im/mcp-docs , https://github.com/scottpersinger
- Cumora: https://cumora.ai , https://github.com/yetone/cumora (README, docs/BYOA.md, docs/COORDINATION.md, docs/PROVIDER_PROFILES.md, docs/LIMITS.md, server/src/tier.ts, src/stores/whispers.ts) , https://github.com/yetone/cumora-releases , https://github.com/yetone , https://trendshift.io/repositories/163884 , https://zentor.ai/blog/what-is-cumora (unverified) , https://4geeks.com/en/blog/ai-tools/cumora (unverified)
