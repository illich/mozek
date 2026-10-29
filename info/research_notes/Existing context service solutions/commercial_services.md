# Commercial / hosted services vs. the "Zuri Context Service" (buy vs. build)

Research date: 2026-10-29. ~25 tool calls. Primary vendor docs were fetched for Guru, Dust, Glean and Anthropic (Google Docs integration). Everything else comes from search snippets or third-party pages and is marked as such. Several candidates in the brief were **not investigated** (see Gaps at the end of each section).

Zuri requirements used below:
- R1: per-file Drive picking + M:N doc↔project
- R2: one owner/maintainer per doc, notified about AI-found inconsistencies
- R3: auto-synced Markdown mirror with diff/history
- R4: auto-rebuilt per-project summary
- R5: remote MCP custom connector in Claude.ai Team, per-user OAuth, per-project roles

## Q1: Does any product cover all five requirements?

### Takeaway
No. No product I checked covers all five. The two closest hosted options are **Dust** (Spaces, remote OAuth MCP server, Drive sync with Sheets, EU hosting, published per-seat price) and **Guru** (owner/verifier workflow with Slack/email notifications, permission-aware remote OAuth MCP approved for Claude). Neither offers a Markdown mirror with diffs (R3) or an automatic per-project summary that is rebuilt on change (R4). Neither turns AI-found contradictions into maintainer notifications (R2 as designed).

### Cited Findings
**Dust (dust.tt)**
- Dust's Google Drive connector lets admins "granularly select the exact data they want to make available" through a selection modal. The docs don't say whether a single file can be picked, as opposed to a folder. — [Dust docs: Google Drive](https://docs.dust.tt/docs/google-drive)
- Synced types: Google Docs, Slides, Sheets (each tab processed separately, up to 50,000 rows per sheet, 128 MB limit), .docx, .pptx, .md, and PDFs with text (no OCR). Files with more than 2 MB of extracted text are ignored. — [Dust docs: Google Drive](https://docs.dust.tt/docs/google-drive)
- **Dust MCP Server** (remote) runs at `https://dust.tt/mcp` (US) and `https://eu.dust.tt/mcp` (EU). It uses OAuth with Dynamic Client Registration, and the client acts as the signed-in user, seeing only the Spaces and data that user can access. Tools: identify the user, list agents, list/create conversations, Pods, files, and "search across workspace Spaces". It "does not proxy third-party tools". Admins can disable it. The page doesn't name Claude.ai explicitly. It requires clients that send the OAuth `resource` parameter. — [Dust MCP Server docs](https://docs.dust.tt/docs/dust-mcp-server)
- Pricing (EUR, excl. VAT): Pro seat €24/month billed yearly (€30 monthly) with 8,000 credits; Max seat €120/month yearly. The Business plan includes 5 remote MCP servers, up to 3 connectors and 5 Spaces. US or EU data residency is offered on Business and Enterprise. SOC 2 Type II is listed. — [Dust pricing](https://dust.tt/home/pricing)

**Guru (getguru.com)**
- Remote MCP server at `https://mcp.api.getguru.com/mcp`. OAuth is "recommended for Claude, Cursor & ChatGPT" and "currently approved for: Claude, Cursor". — [Guru help: connecting MCP](https://help.getguru.com/docs/connecting-gurus-mcp-server-to-claude-desktop) (search snippet)
- MCP calls follow Guru permissions: Knowledge Agent access, source-level permissions, and inherited collection and card permissions. Tools: ask Knowledge Agents, search cards and sources, browse collections, folders and cards, create cards and drafts, move and archive cards, and run usage reports. Google Drive is listed as a connected Source that MCP can reach. Claude.ai, Desktop and Cowork need an Owner to allow network egress for image uploads. — [Guru help: Connecting Guru's MCP Server](https://help.getguru.com/docs/connecting-gurus-mcp-server)
- Verification is Guru's version of a per-document owner. Each card has a verifier (the creator by default) and an interval (default 1 month). When a card is due, Guru notifies the verifier via Slack, email or the web app. Admins can change verifiers in bulk. Only Author, Owner or Admin roles can verify. — [Guru features: verification](https://www.getguru.com/features/verification); [Guru blog: introducing verification](https://www.getguru.com/blog/introducing-verification); [Guru community](https://community.getguru.com/q-a-5/changing-verifier-6536)
- The pricing page shows no prices ("custom", book a call). Its AI Knowledge Layer lists "100+ integrations and MCP delivery" and "Verification workflows and usage signals". EU residency is not mentioned. — [Guru pricing](https://www.getguru.com/pricing)
- Third-party reports put the self-serve price at about $25/seat/month annual ($30 monthly) with a 10-seat minimum. CostBench reports different, conflicting numbers. — [Featurebase](https://www.featurebase.app/blog/guru-pricing); conflicting: [CostBench](https://costbench.com/software/knowledge-management/guru)

**Glean**
- Glean's MCP server supports Claude for Teams/Enterprise (web), Desktop and Code. Auth is OAuth 2.1 with PKCE through Glean's own Authorization Server via SSO (Google Workspace supported), with dynamic client registration. Results are permission-aware and enforced per user. Tools: search, chat, read document, agents, and an MCP gateway. MCP usage is billed through **FlexCredits** (usage-based). — [Glean docs: MCP](https://docs.glean.com/administration/platform/mcp/about)
- Glean doesn't publish prices. Buyer-reported figures are about $50–75/user/month with a commonly cited 100-seat minimum and an annual contract of about $50–60k. These come from competitor blogs, so they may be biased. — [GoSearch](https://www.gosearch.ai/blog/glean-pricing-explained); [eesel](https://www.eesel.ai/blog/glean-ai-pricing); [Workativ](https://workativ.com/blog/glean-pricing)

**Anthropic native (Claude Team)**
- Google Docs added to chats or Project knowledge "sync directly from Google Drive", so the latest version is always used. The Drive connector can read Sheets, Slides, PDFs and Office files. **The Drive connector is available only in private projects and is disabled for shared projects.** An Owner must enable Workspace connectors for the organization. — [Claude support: Google Docs integration](https://support.claude.com/en/articles/10166901-using-the-google-docs-integration)
- Custom remote-MCP connectors are available on every plan, and Team/Enterprise Owners add them for the organization. This comes from third-party guides citing Anthropic support. — [StackOne](https://www.stackone.com/blog/best-mcp-gateway-for-claude/); [Calmara](https://calmara.app/blog/how-to-install-mcp-server-claude-desktop)

**Notion**
- The Notion MCP connector is in Claude's connector directory and works on any Notion plan. AI search across connected sources (including Google Drive) requires Business or Enterprise with Notion AI. The Notion AI connector for Google Drive needs a Notion workspace owner who is also a Google Workspace admin. Business pricing is quoted at $15–20/user/month and sources conflict. — [Notion: AI connector for Google Drive](https://www.notion.com/ja/connections/notion-ai-connector-for-google-drive); [eesel](https://www.eesel.ai/blog/notion-ai-google-drive-connector); [Claude marketplace: Notion](https://claude.com/ko/marketplace/connectors/notion)

**Hjarni**
- Hjarni is a hosted notes and knowledge base with a remote MCP that Claude.ai adds as a custom connector (`https://hjarni.com/mcp`, log in on connect). Pricing: free (25 notes), Pro €9/month, Teams €12/seat/month. It is a notes app with **no Google Drive sync**. — [Hjarni pricing](https://hjarni.com/pricing); [Hjarni Claude docs](https://hjarni.com/docs/claude)

**Needle**
- The Needle MCP documented on GitHub/Glama is a **local** server that uses `NEEDLE_API_KEY` (collections: create, add document by URL, search, list). — [Glama: needle-mcp](https://glama.ai/mcp/servers/needle-ai/needle-mcp)
- `docs.needle-ai.com` now redirects to `needle.app`. That homepage describes an MCP product for Claude web and Cowork priced at "$69/month per connected account, lead lists included", which looks like a pivot to sales and recruiting. Whether this is the same product as the RAG "collections" service is **unverified**. — [needle.app](https://needle.app/)

**Ragie**
- Ragie has a Google Drive connector, and its MCP server exposes a single `retrieve` tool (third-party listing). It is a developer RAG API, not an end-user curation product. — [Ragie: Google Drive connector](https://ragie.ai/connectors/google-drive); [FlowHunt: Ragie](https://www.flowhunt.io/integrations/ragie/)

**fast.io**
- Offers "Cloud Sync" of Google Drive folders into an MCP-accessible workspace (vendor content marketing). Folder-level only, per the snippet. — [fast.io](https://fast.io/resources/sync-google-drive-to-mcp.md)

### Inferences
- **R1 (per-file + M:N):** Not cleanly met by any product. Dust syncs a selection into a connection and then assigns that data to Spaces. Single-file granularity is unconfirmed, and putting the same file into several Spaces probably works but is unverified. Guru syncs Drive as a "Source" without per-file curation evidence. Glean indexes everything the user can access, the opposite of "curated, not crawled". Claude Projects allow per-file picking, but only in private projects.
- **R2 (maintainer + notifications):** Guru's verifier is the closest fit (owner, Slack/email reminders, bulk reassign), but it is time-based re-verification of Guru cards. It is not AI-detected contradictions in Drive files. Dust could approximate it with a custom agent plus a Slack action, but that is a build on Dust.
- **R3 (Markdown mirror with diffs):** Not offered by any product found. Vendors keep opaque vector indexes, and none exposes revision diffs of Drive files via MCP.
- **R4 (auto per-project summary):** Not offered out of the box. Dust or Glean agents could generate one on demand, and Dust's scheduled agents/Pods might do it periodically (unverified).
- **R5 (Claude.ai connector, per-user OAuth, roles):** Dust, Guru and Glean all have OAuth remote MCP with per-user permission enforcement. Dust Spaces and Guru collections roughly match "projects with roles". Guru is explicitly approved for Claude. Dust should work but Claude.ai compatibility is not stated.
- **Billing:** All three add per-seat or usage cost on top of Claude Team: Dust about €24 × 30 ≈ €720/month (~€8.6k/year), Guru about $25 × 30 ≈ $750/month (if the third-party price holds), Glean ≥ ~$50k/year minimum. The in-house design is estimated at €10–40/month of hosting plus small API spend, plus developer time.
- **Double-AI risk:** With Dust and Glean, much of the value is their own chat and agents (with their own model costs and credits). Used only as an MCP back end for Claude.ai, Zuri would pay for features it doesn't use.
- **Cheapest buy-ish path:** Claude Team's native Drive picking works only in private projects. A shared, curated set needs either the GitHub-mirror route (as in Phase 0 of the design) or a custom connector.

### Gaps
- Whether Dust's Drive modal allows **individual file** selection and putting one file into multiple Spaces. Needs a trial or Dust support.
- Whether Dust MCP has been tested as a Claude.ai custom connector. Docs are silent.
- Guru's official prices, EU residency, and how Drive "Sources" relate to verification. Not on the pricing page.
- Glean's per-file curation. Not researched, and its architecture is crawl-everything-permissioned.

## Q2: EU hosting and SOC 2 by vendor

### Takeaway
Only Dust confirms EU data residency (with an EU MCP endpoint) at a published price. Guru and Glean need a sales conversation.

### Cited Findings
- Dust: US or EU data residency on Business and Enterprise, EU MCP at `eu.dust.tt/mcp`, SOC 2 Type II listed. — [Dust pricing](https://dust.tt/home/pricing); [Dust MCP docs](https://docs.dust.tt/docs/dust-mcp-server)
- Guru: the pricing page mentions only a GDPR crosswalk, not EU hosting. — [Guru pricing](https://www.getguru.com/pricing)
- Hjarni: prices in EUR, but hosting location not verified. — [Hjarni pricing](https://hjarni.com/pricing)

### Inferences
- If EU-only hosting is a hard requirement (design open question 2), Dust is the only checked vendor that confirms it publicly.

### Gaps
- Glean and Guru EU residency, and Notion's EU residency for Business. Not checked.

## Q3: Candidates not investigated (no findings)

### Takeaway
Not covered within the time budget: Graphlit, Vectara, Langdock, ContextStream, PuppyOne, Supermemory, Contextual AI, Pinecone Assistant MCP, Slite, Tettra, Atlassian Rovo MCP, Box AI MCP, Egnyte, and any listing of knowledge providers in Anthropic's connector directory.

### Cited Findings
- A Langdock-specific search returned no relevant results.

### Inferences
- From general product category (not verified here): Graphlit, Ragie, Vectara, Pinecone Assistant and Contextual AI are developer RAG APIs. They could replace the sync and search layer of the in-house build, but not the registry, maintainer workflow, or roles. Confluence/Rovo, Box, Egnyte, Slite and Tettra require moving the content into their own store, which goes against "Drive stays the master".

### Gaps
- All items listed above. The highest-value follow-ups are Langdock (EU-based, may have an MCP server and Drive folders) and Graphlit (feeds with summaries and an MCP server).
