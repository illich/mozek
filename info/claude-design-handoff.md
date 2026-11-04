# Design brief: Hub, a team chat built around hand-picked project documents

*Working name "Hub"; the final product name and brand are not decided yet. Use a neutral placeholder wordmark (currently "Hub" set in type, top right of the top bar).*

*Revision 2 (4 Nov 2026): updated after the first design review of the main conversation screen and the Files screen. Changes against revision 1 are summarised in §9; decisions are folded into the sections below. Designs: Design canvas "Hub" (screens Main and Files) and the "Hub" design system (tokens, brand book, MainScreen and FilesScreen).*

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
| **Workspace** | One customer company. Fully isolated from other workspaces. | Workspace name and logo in the top bar (left, above the left column) and in Settings; switching workspaces is rare but possible (user icon menu). |
| **Project** | A unit of work with its own people, documents and rooms. Has a **name, colour and emoji**. | The colour + emoji is the project's identity everywhere: rail tile, a 3px stripe on top of every view header, header tile, "also in" tiles, mentions. |
| **Role** | Per project: **admin**, **member**, **reader**. | Shown once, in the left column's project header ("Admin · 14 members"); not repeated in view headers. Hide or disable actions the role can't do. |
| **Document** | A picked Google Doc, Sheet or other file. Can belong to **several projects** but is synced once. | "Also in" column of small project tiles (emoji on colour, "+N" when more). |
| **Maintainer** | Exactly one person per document. | Avatar + name next to every document ("You" for your own); "Maintained by you" quick view in Files. |
| **Sync status** | Fixed vocabulary: **fresh** (copy matches Drive), **outdated** (Drive changed, update pending), **not fetched** (just added), **error** (export failed: file too large, access lost, unsupported type). | Small coloured status pill followed by its time ("fresh 14:02"), or a short reason where a time alone says nothing ("outdated Drive changed 11:20", "not fetched added 13:12", "error file too large (62 MB)"). Keep these four words exactly. |
| **AI access level** | Per document: **read**, **suggest** (AI may propose edits as Google Docs suggestions or comments), **write**. | **Not a column** in the Files table. Shown and changed (admins) in the row "…" menu and on the document detail page. |
| **Proposal** | A member asks to add a Drive document to a project; an admin approves or rejects it. | "N to approve" chip on Files in the left column; a proposals card on top of Files (§4.3). |
| **Issue** | An inconsistency or problem reported against a document, by the AI (from summaries or from a chat) or by a person. States: open → *fixed* / *not an issue* / *discuss*. | Amber "1 issue" tag after the document title in Files; issue cards quoting both passages; contradiction card inside AI answers (§4.2). |
| **Project summary** | One living Markdown document per project: purpose and status, key decisions (each with its source document), components and shared dependencies, open questions and known inconsistencies, recent changes, and a document map ("what to read for which topic"). | Readable page in the project, with "rebuilt 2 h ago" and a "what changed" section. A dot on Summary in the left column when it changed since you last looked. |
| **Conversation** | A room or chat (kinds below). | Minimal header: project tile, name, topic line, "…" menu. Kind is shown by its icon in the left column (and by the whisper banner inside AI-free rooms). |

### Conversation kinds and visibility
Visibility is shown by a **kind icon in the conversation list**. View headers carry no kind badge (decided in review: too busy); AI-free rooms additionally get the whisper banner inside the room.

| Kind | Icon | Who | AI |
|---|---|---|---|
| 🔒 **Personal** | lock | One person + the AI | Always on, answers every message |
| 👥 **Project room** | `#` | Project members (a project can have several rooms/topics) | Answers when asked (Ask AI button or `@Claude`) |
| 🧩 **Group** | two people | A custom set of people, optionally tied to a project | Answers when asked |
| 🤫 **Whisper** | crossed-out bubble in a dashed box | Usually two humans | **AI off, and guaranteed**: never sent to any model, excluded from search, summaries and anything the AI reads. Distinct dashed `whisper` treatment and banner. |
| 💬 **DM** | bubble | Human-to-human direct messages | AI off, same guarantee as whisper |

**Late invites:** when someone is added to a group, the inviter chooses **"show history: from the start / from now"**. The AI only uses messages that *all current members* may see, so a late joiner can't get hidden history out of the AI. The UI should explain this briefly in the invite dialog.

## 3. Basic screen layout

The layout follows chat tools people already know (Slack, Discord, Teams). Three areas inside a dark frame:

```
┌────┬──────────────────────────────────────────────────────────────────────┐
│    │ N Northwind Aero   [ 🔍                              ]          Hub  │ top bar
│    ├───────────────────┬──────────────────────────────────────────────────┤
│ 🏠 │ 🛰 Ground station  │ ▔▔▔▔▔▔▔▔▔▔▔▔▔ project-colour stripe ▔▔▔▔▔▔▔▔▔▔▔▔ │
│ 💬 │ Admin · 14 members │ 🛰 #power-budget                              … │
│ ── │                   │    Ground station · topic                        │
│ 🛰◀│  📄 Files  2 · 48  │  ───────────────────────────────────────────     │
│ 🚁 │  📝 Summary  •     │                                                  │
│ ✈️ │  ⚑ Issues (3)     │   conversation, full width                       │
│ 📡 │  ── PINNED ──      │   (messages, AI answers streaming in,            │
│ 🔧 │  # general         │    "read 3 documents ⌄", contradiction cards)   │
│ +4 │  ── CONVERSATIONS +│                                                  │
│    │  # power-budget    │                                                  │
│    │  👥 Ops + vendors  │                                                  │
│ ⚙️ │  🔒 My AI chat     │  [ message …            📎 @ 📄 ☺   Send  Ask AI ]│
│ 👤 │  🤫 Petr, Jana     │                                                  │
└────┴───────────────────┴──────────────────────────────────────────────────┘
 rail   left column (264px)  main pane (rest)
```

### 3.1 The frame: icon rail + top bar
The rail and top bar form a **near-black frame in both themes**; the left column and main pane sit inside it as one light panel with a rounded top-left corner.

**Top bar** (48px), one line: workspace mark and name (left, above the left column), the **search field** (magnifier icon only: no placeholder text, no keyboard-shortcut hint), and the **Hub wordmark** at the far right. Search covers the user's conversations and the mirrored documents they can access (never whisper or DM content from other people). Results are grouped: messages, documents, people, projects.

**Left icon rail**, starting **below the top bar** (the top-left corner stays empty). Every item is a 40px tile. Top to bottom:
- **Home**: an overview of what is happening everywhere the user has access (see §4.1).
- **DMs**: *decided:* its own rail icon, directly under Home. DMs aren't tied to a project, and keeping them out of Home lets Home stay a feed.
- a separator;
- **Project tiles**: the project's emoji on its colour. **Sorted by that user's own usage**, not alphabetically. More than five projects: a dashed **"+N"** tile gathers the rest (popover list with search).
- *(flexible empty space)*
- **Settings**: second from the bottom.
- **User**: the user's round avatar, at the very bottom (profile, notification preferences, sign out, switch workspace).

**Active project:** a small triangle at the rail's right edge, in the left column's colour, pointing into the left column. Nothing is drawn on the rail's left edge.
**Badges:** red counts for mentions only. Plain unread activity is not shown on the rail.

### 3.2 Left column (264px, resizable 220–360px)
The list for the selected rail item:
- **In a project:**
  - Project header: emoji tile, name, "Admin · 14 members" (the only place showing role and member count); click for the project menu.
  - Fixed items **Files** ("N to approve" chip for admins, document count), **Summary** ("rebuilt 2 h ago", dot when changed since last look), **Issues** (count, "1 on your docs").
  - **Pinned** section.
  - **Conversations**, most recent activity first, with "+" to create a room, group or personal chat, and "N older" to expand the rest.
  - Row cues: kind icon; bold = unread, grey = read; on the right a red count for mentions of you, otherwise last-activity time; a pulsing orange square when the AI is answering there right now.
- **In Home:** feed sections and the user's personal chats across projects.

### 3.3 Main pane (wide)
Usually **one conversation**. Otherwise the selected view: Files, Summary, Issues, a document detail, settings pages. **Content uses the full width of the pane** (24px side padding, no max-width).

**View header** (every view): a 3px project-colour stripe, the project tile, the room or view name, one muted line of context under it (project · topic, or e.g. "48 documents. Claude reads only these."), and on the right only the main action (if any) plus a **"…" menu**. *Decided in review:* no member avatars, kind badge, AI on/off pill, model name or role chip in the header; that information lives in the left column or in the "…" menu (room details: members, AI model, visibility).

## 4. Main screens and features

### 4.1 Home
- An activity feed across all of the user's projects: new messages and mentions, documents synced or changed, issues opened against the user's documents, summaries rebuilt, proposals waiting for approval, people joining.
- "Needs your attention" at the top: open issues on documents the user maintains, proposals to approve (admins), unread mentions.
- Never shows other people's whisper or DM content.
- *Not designed yet.*

### 4.2 Conversation *(designed: MainScreen)*
- Markdown messages, reactions, mentions of people and of `@Claude`. **No threads in the initial version.**
- **Composer:** two buttons replace the send arrow: **Send** (teal-green, Hub's accent) posts to the room and the AI stays quiet; **Ask AI** (orange, the AI colour, spark icon) posts and asks the AI to answer. Typing `@Claude` still works. A hint left of the buttons says how many documents the AI can read here ("Claude can read 48 documents in Ground station").
- **AI messages:** rounded-square AI avatar (humans are round), name in the AI colour, an "AI" tag, then on the same line "14:01 · replying to Petr · **read 3 documents ⌄**". The document chips (numbered, matching inline citation numbers) are **hidden by default** and shown by clicking the chevron. The signature line is the same height as a human author's line.
- **AI answers stream in.** While answering, a pulsing "Reading Power budget v3" chip, a caret and a Stop button.
- **Contradiction card** inside an AI answer: both passages side by side (document, section, sync status, maintainer, quoted value in mono), one line of context, then **Dismiss / Discuss / Report to {maintainer}**. "Discuss" continues in the room (no threads).
- Mentions of the viewer: the whole message on a soft yellow background, the @name in the mention colour.
- **Attachments** (files, images) can be dropped into any message and are passed to the AI.
- **Long conversations** work like a normal Claude chat. Older parts are summarized automatically; a dashed hairline marks it: "214 earlier messages are summarized for Claude · Show summary".
- The AI can propose "report this as an issue to the maintainer" when it spots a contradiction, and "promote to doc": turn a conversation outcome into a Google Doc draft or a comment on an existing document.
- Invite dialog with the **from the start / from now** history choice.
- Whisper rooms: clear "AI is off. Nothing here is ever sent to an AI model" banner, replacing the composer hint and the Ask AI button.

### 4.3 Files (per project) *(designed: FilesScreen)*
- **Header:** "Files", "Ground station · 48 documents. Claude reads only these.", **Add from Drive** (admins; members see **Propose a document**; readers neither), "…" menu.
- **Proposals card** (admins only, above the list): "2 documents proposed for this project", **one line per proposal**: type icon, title, "proposed by Petr Novák, maintained by Tomáš Suchý" (or just the proposer when they are also the maintainer), date, **Reject** and **Approve…**. Approve opens a step to confirm maintainer, AI access and projects, and shows the proposer's reason.
- **Toolbar:** segmented quick views **All · Maintained by you · With open issues · Not fresh** (with counts), an icon-only search field, and one **Filter** button for everything else (type, maintainer, project…). Sorting by clicking column headers.
- **Table, one line per document** (44px rows):
  - **Document** (about half the width): type tile (Doc blue, Sheet green, PDF red) + title linking to the document detail. No folder path. Open issues as an amber **"1 issue"** tag right after the title; when the title is too long it truncates with "…" and the tag stays visible at the column's right edge.
  - **Maintainer**: avatar + name ("You").
  - **Sync**: status pill + time or short reason (see §2).
  - **Also in**: small tiles of the other projects, "+N".
  - Row **"…" menu**: open in Drive, change maintainer, AI access (read / suggest / write), projects, remove.
- **Infinite scroll:** the first 50 documents load initially, the rest as you scroll. No "Showing N of M" line.
- Actions by role: admins **add documents** through the Google Drive file picker (pick files, then set maintainer, AI access and projects), remove them, and change the maintainer; members **propose** a document (paste a Drive link + reason).
- A **document detail** view shows its Markdown copy, recent changes (diff), issues, AI access and the projects it belongs to. *Not designed yet.*

### 4.4 Summary (per project)
A readable rendering of the living project summary, with the rebuild time, a "what changed since you last looked" section, and links from every statement to its source document. Admins can trigger a rebuild; a maintainer can mark a document change as "major" to force one. *Not designed yet.*

### 4.5 Issues
- Per project, plus a personal **"My documents"** view for maintainers (their documents and the issues against them).
- An issue card shows the quoted passage, the conflicting document and passage, why the AI thinks they conflict, where it came from (summary run, chat link, or a person), and the buttons **Fixed / Not an issue / Discuss** ("Discuss" opens a conversation pre-filled with the context).
- Maintainers also get a weekly digest by email.
- *Not designed yet.*

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

- **Always know where you are and who can see it.** Project colour + emoji (rail triangle, header stripe and tile), room name, topic and the kind icon in the list are always visible. Visual cues matter more than in a normal chat app, because the AI's answers depend on which project and visibility you are in.
- **Less on screen.** *(added in review)* One line per row; no labels that repeat what an icon or the context already says; no counters nobody acts on; optional details go into the "…" menu, a tooltip or an expandable line (e.g. "read 3 documents ⌄").
- **Familiar over novel.** People switch from Slack and Claude.ai, so keep the three-area layout and common chat conventions.
- **AI as a polite participant, not a takeover.** It answers when asked (except in personal chats), says which documents it read, and cites sources.
- **Ownership is visible.** Maintainers' faces appear next to documents and issues; issues feel like helpful notes, not alarms.
- **Trust for private talk.** Whisper and DM rooms must look and feel clearly AI-free.
- **Calm, professional, dense enough for power users.** Light and dark themes (the frame stays dark in both). Fully usable on a phone (rail collapses to a bottom bar: Home, DMs, Projects, You; left column and main pane become stacked screens).
- **White-label-ready.** Hub is a product sold to other companies: no customer-specific branding baked into the UI; each workspace can set its name and logo.

## 8. Open points

Resolved in the first review:
- ~~Whether DMs get their own rail icon or live inside Home.~~ Own rail icon under Home (§3.1).
- ~~Ordering and grouping in the left column.~~ Pinned section, then conversations by recent activity, "N older" to expand (§3.2).
- ~~Exact Files table columns and filters.~~ See §4.3.
- ~~The visual identity, since the product name is not final.~~ First pass in the "Hub" design system: dark frame, Geist type, teal-green accent, orange for everything the AI does; placeholder "Hub" wordmark.

Still open:
- The exact content and grouping of the Home overview.
- How the summary page highlights changes since the last visit.
- The mobile navigation pattern in detail (bottom bar proposed).
- Behaviour of "Discuss" on a contradiction card now that there are no threads (proposal: quote the card into the composer).
- What clicking an inline citation number does while the document list is collapsed (open the document, or expand the list).
- Where the proposer's reason is shown (proposal: in the Approve step, and as a tooltip on the proposal row).
- Whether the conversation kind needs any cue in the room itself besides the whisper banner, now that the header badge is gone.

## 9. Changes in revision 2 (first design review, 3–4 Nov 2026)

- Frame (rail + top bar) is near-black in both themes; the work area is a light panel inside it.
- Top bar: workspace, icon-only search (no placeholder, no shortcut hint), "Hub" wordmark top right.
- Rail starts below the top bar; all items 40px tiles; active project marked by a triangle on the right edge; no left-edge markers; badges for mentions only.
- View headers reduced to stripe, tile, name, context line, main action and "…" menu. Removed: member avatars, kind badge, AI pill, model name, role chip. Role and member count only in the left column.
- Composer: **Send** and **Ask AI** buttons.
- AI signature line carries "read N documents ⌄"; document chips collapsed by default.
- No threads in v1; "Discuss in thread" became "Discuss".
- Conversation content uses the full pane width.
- Files: one-line rows; no folder path; issues as a tag after the title; no AI access or issues column; sync pill + bare time; title column ~50 %; quick views + icon search + one Filter button; no stats line; one-line proposals; infinite scroll (50 rows, then on scroll).
