# Design brief: Hub, a team chat built around hand-picked project documents

*Working name "Hub"; the final product name and brand are not decided yet. Use a neutral placeholder wordmark.*

## 1. What the product is

Hub is a web app (desktop first, also usable on phones as an installable web app) for small and mid-sized companies. It is sold as a public service at roughly **$5 per user per month**.

It combines two things:

1. **A curated knowledge layer.** For each project, an admin hand-picks the 10–100 Google Drive documents and spreadsheets that matter. The AI only ever sees those documents, never the whole Drive (company drives are full of old and obsolete files, and broad access is a security risk). Hub keeps a Markdown copy of every picked file, re-syncs it automatically when the original changes, and maintains one living **project summary** that is rebuilt as documents change.
2. **A team chat where the AI is a participant.** People chat with each other and with the AI (Claude by default; other models, including local ones, can be plugged in) inside project rooms. The AI can read every document of the project, cite it, and answer questions across all of them. It replaces Slack for the companies that adopt it.

The core idea: **every document has exactly one human maintainer.** When the AI finds a contradiction (say, the ground-station spec says 12 V and the drone spec says 24 V), it raises an **issue** to that document's maintainer, who decides what to do. Owned, curated documents plus an AI that flags inconsistencies is what sets Hub apart from generic "AI search over everything" tools.

Customers can also use their project documents from their **own Claude subscription**: Hub provides a connector that a company adds to Claude.ai once, after which Claude.ai can read the same curated documents with the same permissions. This is a selling point ("makes the Claude you already pay for know your project documents"), and it shapes the billing model (see §6).

### Who uses it
- **Typical customer:** a technical company of 10–200 people with a few large projects and many small ad-hoc ones. Projects overlap heavily: one document (a shared component spec, say) often belongs to several projects.
- **Workspace owner / billing admin:** signs the company up, connects Google, invites people, pays.
- **Project admin:** adds people and documents to a project and approves document proposals.
- **Project member:** chats, reads, and proposes documents to add.
- **Project reader:** read-only access.
- **Document maintainer:** any person who owns one or more documents and receives the AI's issue reports about them.

## 2. Core concepts (the vocabulary the UI must make visible)

| Concept | What it is | UI implications |
|---|---|---|
| **Workspace** | One customer company. Fully isolated from other workspaces. | Workspace name and logo in Settings; switching workspaces is rare but possible (user icon menu). |
| **Project** | A unit of work with its own people, documents and rooms. Has a **name, colour and emoji**. | The colour + emoji is the project's identity everywhere: rail icon, room headers, chips, mentions. |
| **Role** | Per project: **admin**, **member**, **reader**. | Show the viewer's role in the project header; hide or disable actions the role can't do. |
| **Document** | A picked Google Doc, Sheet or other file. Can belong to **several projects** but is synced once. | Show "also in: 🛰 Ground station, 🚁 Small drones" on a document. |
| **Maintainer** | Exactly one person per document. | Avatar + name next to every document; "My documents" view for maintainers. |
| **Sync status** | Fixed vocabulary: **fresh** (copy matches Drive), **outdated** (Drive changed, update pending), **not fetched** (just added), **error** (export failed: file too large, access lost, unsupported type). | Small coloured status pill with tooltip and last-synced time. Keep these four words exactly. |
| **AI access level** | Per document: **read**, **suggest** (AI may propose edits as Google Docs suggestions or comments), **write**. | Icon or label on the document row; changing it is an admin action. |
| **Proposal** | A member asks to add a Drive document to a project; an admin approves or rejects it. | Pending-proposals badge for admins. |
| **Issue** | An inconsistency or problem reported against a document, by the AI (from summaries or from a chat) or by a person. States: open → *fixed* / *not an issue* / *discuss*. | Issue cards quoting the passage, the conflicting document and the source conversation, with those three buttons. |
| **Project summary** | One living Markdown document per project: purpose and status, key decisions (each with its source document), components and shared dependencies, open questions and known inconsistencies, recent changes, and a document map ("what to read for which topic"). | Readable page in the project, with "rebuilt 2 h ago" and a "what changed" section. |
| **Conversation** | A room or chat (kinds below). | Clear header: project, kind/visibility, topic, members, whether AI is on. |

### Conversation kinds and visibility
Visibility must be obvious at a glance: a badge in every room header and an icon in the conversation list.

| Kind | Who | AI |
|---|---|---|
| 🔒 **Personal** | One person + the AI | Always on, answers every message |
| 👥 **Project room** | Project members (a project can have several rooms/topics) | Answers when `@mentioned` |
| 🧩 **Group** | A custom set of people, optionally tied to a project | Answers when `@mentioned` |
| 🤫 **Whisper** | Usually two humans | **AI off, and guaranteed**: never sent to any model, excluded from search, summaries and anything the AI reads. Needs a distinct, reassuring visual treatment. |
| 💬 **DM** | Human-to-human direct messages | AI off, same guarantee as whisper |

**Late invites:** when someone is added to a group, the inviter chooses **"show history: from the start / from now"**. The AI only uses messages that *all current members* may see, so a late joiner can't get hidden history out of the AI. The UI should explain this briefly in the invite dialog.

## 3. Basic screen layout

The layout follows chat tools people already know (Slack, Discord, Teams). Three areas:

```
┌────┬──────────────────────────────────────────────────────────────────────┐
│    │  [ 🔍 Search                                    ]           top bar  │
│ 🏠 ├───────────────────┬──────────────────────────────────────────────────┤
│(💬)│ 🛰 Ground station  │  👥 #power-budget · Ground station · topic …     │
│ 🛰 │                   │  ───────────────────────────────────────────     │
│ 🚁 │  📄 Files          │                                                  │
│ ✈️ │  📝 Summary        │   conversation                                   │
│ 🔧 │  ⚠️ Issues (3)     │   (messages, AI answers streaming in,            │
│ 📡 │  ───────────       │    "read doc X" chips, attachments)              │
│ …  │  👥 #general       │                                                  │
│    │  👥 #power-budget  │                                                  │
│    │  🧩 Ops + vendors  │                                                  │
│    │  🔒 My AI chat     │                                                  │
│    │  🤫 Petr, Jana     │                                                  │
│ ⚙️ │                   │                                                  │
│ 👤 │                   │  [ message composer · 📎 · @Claude          ]    │
└────┴───────────────────┴──────────────────────────────────────────────────┘
 rail   left column (narrow)   main pane (wide, about 2/3 of a widescreen)
```

### 3.1 The frame: icon rail + top bar
**Left icon rail (narrow column), top to bottom:**
- **Home**: an overview of what is happening everywhere the user has access (see §4.1).
- **DMs**: possibly its own icon; not decided yet. The designer may propose whether DMs live in the rail or inside Home.
- **Project icons**: one per project the user belongs to, shown as the project's emoji on its colour. **Sorted by that user's own usage**, not alphabetically. With more than five projects, a **"…" icon** gathers projects 6 and up (a popover list with search).
- *(flexible empty space)*
- **Settings**: second from the bottom.
- **User**: the user's avatar, at the very bottom (profile, notification preferences, sign out, switch workspace).

Unread and mention badges go on rail icons.

**Top bar:** one line, holding only a **search field** for now. Search covers the user's conversations and the mirrored documents they can access (never whisper or DM content from other people). Results are grouped: messages, documents, people, projects.

### 3.2 Left column (narrow)
The list for the selected rail item:
- **In a project:** the project name and emoji at the top, then fixed items **Files**, **Summary** and **Issues** (with a count), then the project's conversations in roughly **chronological order** (most recent activity first), each with its kind icon and unread state. A "+" creates a room, group or personal chat.
- **In Home:** feed sections and the user's personal chats across projects.
- Details are open for the designer to propose: grouping, pinning, collapsing.

### 3.3 Main pane (wide)
Usually **one conversation**. Otherwise the selected view: Files, Summary, Issues, a document detail, settings pages.

**Room header** (always visible): project emoji + colour, room name, visibility badge (🔒/👥/🧩/🤫/💬), topic, member avatars, AI on/off indicator, model in use, and the viewer's role.

## 4. Main screens and features

### 4.1 Home
- An activity feed across all of the user's projects: new messages and mentions, documents synced or changed, issues opened against the user's documents, summaries rebuilt, proposals waiting for approval, people joining.
- "Needs your attention" at the top: open issues on documents the user maintains, proposals to approve (admins), unread mentions.
- Never shows other people's whisper or DM content.

### 4.2 Conversation
- Markdown messages, threads, reactions, mentions of people and of `@Claude`.
- **AI answers stream in.** While answering, the AI's document reads are shown as small collapsible **chips** ("read 📄 Power budget v3"), each linking to the document. Sources are cited inline.
- **Attachments** (files, images) can be dropped into any message and are passed to the AI.
- **Long threads** work like a normal Claude chat. Older parts are summarized automatically behind the scenes; a subtle marker can show where.
- The AI can propose "report this as an issue to the maintainer" when it spots a contradiction in a chat, and "promote to doc": turn a conversation outcome into a Google Doc draft or a comment on an existing document.
- Invite dialog with the **from the start / from now** history choice.
- Whisper rooms: clear "AI is off. Nothing here is ever sent to an AI model" banner.

### 4.3 Files (per project)
A table of the project's documents:
- title + type icon (Doc, Sheet, PDF…), with a link that opens it in Google Drive;
- **maintainer** (avatar + name);
- **status pill**: fresh / outdated / not fetched / error, with the last-synced time;
- AI access level (read / suggest / write);
- "also in" other projects;
- open-issue count.

Actions by role: admins **add documents** through the Google Drive file picker (pick files, then set maintainer, AI access and projects), remove them, and change the maintainer; members **propose** a document (paste a Drive link + reason). A document detail view shows its Markdown copy, recent changes (diff), issues and the projects it belongs to.

### 4.4 Summary (per project)
A readable rendering of the living project summary, with the rebuild time, a "what changed since you last looked" section, and links from every statement to its source document. Admins can trigger a rebuild; a maintainer can mark a document change as "major" to force one.

### 4.5 Issues
- Per project, plus a personal **"My documents"** view for maintainers (their documents and the issues against them).
- An issue card shows the quoted passage, the conflicting document and passage, why the AI thinks they conflict, where it came from (summary run, chat link, or a person), and the buttons **Fixed / Not an issue / Discuss** ("Discuss" opens a conversation pre-filled with the context).
- Maintainers also get a weekly digest by email.

### 4.6 Project settings (admins)
Name, emoji, colour, description; members and their roles (admin / member / reader); rooms; default AI model for the project (including a local/private model for sensitive projects); AI spend cap for the project; pending proposals.

### 4.7 Workspace settings (owner / workspace admins)
- **Members:** invite by email or allow everyone from the company's Google domain; deactivate people.
- **Sign-in:** Google (default) and other OpenID Connect providers. No passwords are stored.
- **Google Drive connection:** status of the Drive connection and of the sync service account.
- **AI:** see §6. Choose how AI chat inside Hub is paid for, set API keys, models and spend caps.
- **Claude connector:** step-by-step instructions to add Hub to the company's Claude.ai organisation, plus connection status.
- **Billing:** plan, seats in use vs. paid, trial status, invoices, payment method, AI usage this month (tokens and cost, per project and per user).
- **Data and privacy:** retention settings, audit log, export of all data (the document mirror is an exportable archive), delete the workspace.

### 4.8 User menu
Profile, notification preferences (web push, mobile, email digest), connected sign-in accounts, theme, sign out, switch workspace.

## 5. Public-service flows (new for the public launch)

### 5.1 Sign-up and onboarding
The first ten minutes decide adoption, so this flow deserves the most design care. Target flow:
1. Landing page → **"Sign in with Google"** (or another OpenID Connect provider).
2. Create the workspace: company name, logo (optional), Google domain.
3. Create the first project: name, emoji, colour.
4. **Pick the first documents** with the Google Drive picker; assign maintainers (defaults to the person picking).
5. Invite colleagues (emails or "everyone at @company.com").
6. Optional: **connect Claude.ai** (show the connector setup), and/or set up AI for in-app chat.
7. Land in the project's first room with the AI ready, a suggested first question, and the summary being built (show progress: documents fetched, summary building).

A checklist ("3 of 6 steps done") in Home until onboarding is complete. A free trial runs during this time.

### 5.2 Billing
- Per-seat subscription around $5/user/month, with a free trial, automatic seat counting, invoices, and failed-payment handling.
- Pricing page and in-app billing page should make clear what's included and what AI costs extra (see §6).

### 5.3 Trust pages
Public privacy policy, terms, security page, sub-processor list, EU hosting statement. These live on the marketing site, not in the app, but Settings links to them.

## 6. AI usage and its cost: a visible design concern

The $5 seat price covers the curated knowledge layer, human chat and the included (capped) project summaries. **AI chat costs extra**, and the customer chooses how to pay:
1. **Use your own Claude subscription:** chat in Claude.ai via the Hub connector. No extra cost from Hub.
2. **Bring your own API key** (Anthropic, OpenAI, or a local/private model endpoint) for AI inside Hub.
3. **Hub AI credits:** prepaid or metered, billed on the Hub invoice.

Design implications:
- Settings → AI shows the chosen mode, usage this month, and caps per workspace, project and user.
- Gentle warnings when approaching a cap; when a cap is reached, the AI replies with a clear, non-alarming message and a link for admins.
- Per-message cost is **not** shown to regular users; usage dashboards are for admins.
- Summary rebuilds included in the plan are capped; show remaining summary capacity in admin settings.

## 7. Design principles

- **Always know where you are and who can see it.** Project colour + emoji, room kind badge, topic and members are always visible. Visual cues matter more than in a normal chat app, because the AI's answers depend on which project and visibility you are in.
- **Familiar over novel.** People switch from Slack and Claude.ai, so keep the three-area layout and common chat conventions.
- **AI as a polite participant, not a takeover.** It answers when asked (except in personal chats), shows which documents it read, and cites sources.
- **Ownership is visible.** Maintainers' faces appear next to documents and issues; issues feel like helpful notes, not alarms.
- **Trust for private talk.** Whisper and DM rooms must look and feel clearly AI-free.
- **Calm, professional, dense enough for power users.** Light and dark themes. Fully usable on a phone (rail collapses to a bottom bar or drawer; left column and main pane become stacked screens).
- **White-label-ready.** Hub is a product sold to other companies: no customer-specific branding baked into the UI; each workspace can set its name and logo.

## 8. Open points the designer may propose solutions for

- Whether DMs get their own rail icon or live inside Home.
- The exact content and grouping of the Home overview.
- Ordering and grouping in the left column (pinning, sections, collapsed groups).
- Exact Files table columns and filters.
- How the summary page highlights changes since the last visit.
- The mobile navigation pattern.
- The visual identity, since the product name is not final.
