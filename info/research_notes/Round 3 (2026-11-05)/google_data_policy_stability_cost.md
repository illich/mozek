# Round 3 notes: Google data policy for AI use, API stability, cost (checked 2026-11-05)

Scope: input for `google-drive-grill.md` questions 9, 12 and 13. Dates in brackets are the page's visible "last updated" stamp. (unverified) = third-party only; (inferred) = not stated verbatim.

## Q1. May we feed Workspace data to an AI model?

**Which documents govern.** Two layers: (a) the *Google API Services User Data Policy* [S1] (Feb 15, 2024), generic, no AI clause; (b) the *Google Workspace API User Data and Developer Policy* [S2] (no date stamp), Workspace-specific, contains the AI clause and the approved-use-case lists. The OAuth verification FAQ [S3] restates the AI rule.

**Limited Use, general policy [S1], verbatim.** "Limit your use of data to providing or improving user-facing features that are prominent in the requesting application's user interface"; "Transfers of data are not allowed, except: To provide or improve your appropriate access or user-facing features that are visible and prominent in the requesting application's user interface and only with the user's consent; For security purposes (for example, investigating abuse); To comply with applicable laws; or, As part of a merger, acquisition, or sale of assets of the developer after obtaining explicit prior consent from the user."; "Don't allow humans to read the data, unless: You first obtained the user's affirmative agreement to view specific messages, files, or other data … It is necessary for security purposes … to comply with applicable law; or The data (including derivations) is aggregated and used for internal operations…"; "All other transfers, uses, or sales of user data are prohibited, including: Transferring or selling user data to third parties like advertising platforms, data brokers, or any information resellers …"; "You must ensure that your employees, agents, contractors, and successors comply with this Google API Services User Data Policy."
- Applicability: "Sensitive and Restricted Scopes … are subject to additional requirements that can be found in each product's User Data and Developer Policy" (Appropriate Access, Limited Use, Secure Data Handling). "If your app is only used by users within your own domain, then these requirements do not apply." [S1]
- No "service provider" / sub-processor language exists in either Google document. Sending content to Anthropic is a *transfer*; the only applicable exception is the first one (user-facing feature, visible and prominent, user consent).

**AI clause, Workspace policy [S2], verbatim** (in the prohibited list): "Transferring, selling, or using user data to create, train, or improve a machine learning or artificial intelligence model beyond that specific user's personalized model for the appropriate use case or user-facing feature." FAQ [S3]: "In the context of AI/ML this means Google User Data must be used only for a personalized model"; "Google User Data may not be used to train or improve foundational or frontier models". Workspace blog (Feb 3, 2024) [S4]: developers "will be required to commit via their privacy policies that they do not retain user data obtained through Workspace APIs to develop, improve, or train non-personalized AI and/or ML models." The current policy page [S2] instead requires the privacy policy to contain "An affirmative or other similar statement that your use of the data complies with the Limited Use restrictions". Treat both as required.
- Inference ≠ training. Google's wording targets "create, train, or improve" a model; LLM inference on behalf of the user is a transfer under the user-facing-feature exception (inferred). Conditions: feature visible/prominent in the UI, user consent, no training by the recipient, privacy-policy statements, "employees, agents, contractors" bound.
- Limited Use applies to **Sensitive and Restricted** scopes [S2]: "These requirements apply to data derived from both Sensitive and Restricted scopes." Scope classes [S5][S6][S7]: `drive`, `drive.readonly`, `drive.metadata(.readonly)`, `drive.activity*` = **Restricted**; `documents`, `documents.readonly`, `spreadsheets`, `spreadsheets.readonly` = **Sensitive**; `drive.file` = **Non-sensitive** ("Recommended"). Non-sensitive scopes: no verification required [S8], not subject to the Limited Use additional requirements [S1].
- Approved use cases for Drive restricted scopes [S2]: sync/backup apps; "Productivity and educational applications" that "only use Restricted Scopes to handle Drive files (or their metadata or permissions) via the application's user interface"; reporting/security apps. A knowledge-base Q&A app is arguable under "productivity"; `drive.file` + Picker avoids the question entirely.
- Agentic requirements in [S2] applying to all apps: "granularly require users to confirm the MCP, tool, skill, or other agentic behavior invocation"; "Protecting against prompt injection techniques by either using Google Cloud Platform's Model Armor or other prompt injection protection."
- In-product disclosure [S2]: "You must provide a disclosure of your data access, collection, use, and sharing" that "Must be within the application itself if application-based or in a separate dialog window if web-based; Cannot be placed only in a privacy policy or terms of service". [S1]: "Disclosures about data use should be prominent and timely."
- Marketplace program policies [S9] (Aug 28, 2025): no AI/generative section. Relevant: "Prominently disclose how the user data will be used, and Obtain the user's affirmative consent for such use"; "Limit your use of user data to providing or improving your single purpose".
- "AI-powered agent" OAuth client flag (release notes Oct 12, 2026) [S10]: applies only to Google Workspace MCP servers; "Apps and integrations that are not AI agents, like those that only call Google Workspace REST APIs, are not affected by this change." No policy text attached to the flag was found.
- Anthropic side: Commercial Terms (effective Jun 17, 2025) [S11]: "Anthropic may not train models on Customer Content from Services." Privacy Center (Aug 18, 2026) [S12]: "we will not use your inputs or outputs from our commercial products" to train (API, Claude for Work). **Caveat:** Claude.ai Free/Pro/Max accounts may be used for training when the user's setting is on (5-year retention) [S13] → employees reaching our MCP server from Claude.ai must be on Team/Enterprise (Commercial Terms) or have training off, otherwise Limited Use is breached.

## Q2. Stability

- Notice commitments: Google APIs ToS (Nov 9, 2021) [S14]: Google may "discontinue the APIs … for any reason and at any time"; changes "will become effective no sooner than 30 days after they are posted". Cloud ToS §1.4(e) (Mar 24, 2026) [S15]: "Google will notify Customer at least 12 months before … discontinuing any Service … or … significantly modifying a Customer-facing Google API in a backwards-incompatible manner", but "for GWS Services, only applies to Core Services"; whether Workspace developer APIs fall under this is not established. Concrete commitment for the quota/billing model: "at least 90 days' notice" [S16]. No separate "Workspace APIs deprecation policy" page found.

| Date | Change | Source |
|---|---|---|
| 2019-05-31 | Drive restricted scopes announced; verification from 2020; `drive.file` + Picker apps "will not be required to go through the restricted scope verification" | [S17] |
| 2022-02-28 → 2023-01-31 | OAuth out-of-band (OOB) flow blocked | [S18] |
| 2024-07-11 | New restricted scope `drive.meet.readonly` | [S10] |
| 2024 → 2025-03-14 | Less secure apps / password-based IMAP, CalDAV, Google Sync turned off for all accounts | [S19] |
| 2025-04-15 | New service accounts get no Drive storage and cannot own files (unverified: forum reposts of the Google notice). Admin page (2026-10-26): standalone service accounts get a 403; use a shared drive or domain-wide delegation | [S20] |
| 2025-06 | OAuth client secrets masked; clients inactive 6 months auto-deleted | [S21] |
| 2025-12-01 | Granular OAuth consent mandatory for HTTP add-ons | [S10] |
| 2026-02-25 | `enforceExpansiveAccess` deprecated on `permissions` | [S10] |
| 2026-05-01 | Drive API quotas measured in quota units; 1 TB/day egress per Workspace user; tiering model; existing projects keep quotas ≥60 days | [S10][S16][S22] |
| 2026-05-18 | Drive event subscriptions (Workspace Events API) GA | [S10] |
| 2026-06-27 | ADA CASA page describes "AL1/AL2" assurance levels; Google FAQ still says Tier 2/3 | [S23][S3] |
| 2026-10-05 | Drive/Docs edit `.md` files natively (no API change noted) | [S24] |
| 2026-11-11 | Google Workspace MCP servers reject non-agentic OAuth clients (REST unaffected) | [S10] |
| Later 2026 | Over-threshold API usage billed; quota increases require billing; 90 days' notice | [S16][S22] |
| — | Drive v2: no sunset announced; v3 is current. No `drive.file`/Picker semantic changes found 2024–2026. | [S25] |

- Watch: Workspace developer release notes [S10]; Workspace Updates blog; tools-safety page [S16]; Cloud deprecations (cloud.google.com/terms/deprecation); Google Developers blog (Auth Platform posts) [S21].

## Q3. Cost

- Drive/Docs/Sheets APIs: free today. Drive limits [S22]: "All standard use of the Google Drive API is available at no additional cost." / "Exceeding the quota request limits is planned to incur charges to your Google Cloud billing account later in 2026." / daily threshold "400,000,000 quota units"; "Usage under this threshold doesn't incur extra charges and your Google Cloud account isn't billed." / "Full billing details will be shared later in 2026 with at least 90 days' notice before any changes take effect." Per method: read 5, list 100, edit 50, download 200 units; 1,000,000 units/min/project, 325,000/min/user. Docs [S26]: 3,000 read / 600 write per min per project; Sheets [S27]: 300/min project, 60/min user; both "no additional cost", same "planned … later in 2026" sentence. **No price list or announcement published as of today** (tools-safety page: paid option "coming soon", "less than 1% of active developers will need to move beyond the standard usage tier").
- Billing account: not required now to enable Drive/Docs/Sheets or the Events API (inferred: "Later in 2026, increase requests will require Google Cloud billing to be enabled" [S16]; the Pub/Sub quickstart lists no billing prerequisite [S28]). Pub/Sub [S29]: "the first 10 GiB of throughput … for a billing account is free. After that, the price is $40 per TiB". Cloud Run free tier [S30]: 2M requests, 180,000 vCPU-s, 360,000 GiB-s per month. The Workspace Events API "supports Google Cloud Pub/Sub topics as a notification endpoint" [S35] and offers Drive `file.v3.contentChanged` etc. [S31].
- OAuth verification: free. CASA [S3]: "Google does not charge the developer any fees for security assessment."; "The cost for such a service is agreed on between the developer and the assessor without any involvement from Google."; "must be done once a year". Needed only for restricted scopes when data is stored/transmitted server-side [S5]; "You won't be required to get a security assessment for projects with no restricted scopes" [S3]. Vendor prices (unverified): TAC Security Tier 2 Basic about $540/app/yr, Premium $720–855, Tier 3 about $4,500 [S32]; the 2019 figure "$15,000–$75,000" is obsolete press.
- Marketplace [S33]: "Google doesn't charge a listing fee to distribute apps through the Marketplace." (older Developer Terms text mentions a "Listing Fee"; which governs is unverified.) Developer Preview Program [S34]: no fee mentioned; needs a Workspace account + Cloud project.

## Practical reading
- Internal phase: the own-domain exemption ⇒ the Limited Use additional requirements "do not apply"; restricted scopes still need the Internal user type, no CASA (verify).
- Commercial phase: primary path = `drive.file` only (listed as "Recommended Non-sensitive" for the Drive, Docs and Sheets APIs) → no verification beyond brand, no CASA, no Limited Use additional requirements. Fallback: `documents.readonly`/`spreadsheets.readonly` (sensitive → verification, no CASA). Avoid `drive`/`drive.readonly` (restricted → CASA yearly). Ship an in-app disclosure + consent screen, privacy-policy Limited Use + no-AI-training statements, Anthropic API / Team / Enterprise only, user confirmation for MCP tool invocations, prompt-injection mitigation.

## Sources
- S1 https://developers.google.com/terms/api-services-user-data-policy (Feb 15, 2024)
- S2 https://developers.google.com/workspace/workspace-api-user-data-developer-policy
- S3 https://support.google.com/cloud/answer/13463817
- S4 https://workspace.google.com/blog/ai-and-machine-learning/api-policy-protections (Feb 3, 2024)
- S5 https://developers.google.com/workspace/drive/api/guides/api-specific-auth
- S6 https://developers.google.com/workspace/docs/api/auth
- S7 https://developers.google.com/workspace/sheets/api/scopes
- S8 https://support.google.com/cloud/answer/9110914
- S9 https://developers.google.com/workspace/marketplace/terms/policies (Aug 28, 2025)
- S10 https://developers.google.com/workspace/release-notes
- S11 https://www.anthropic.com/legal/commercial-terms (eff. Jun 17, 2025)
- S12 https://privacy.claude.com/en/articles/7996868-is-my-data-used-for-model-training (Aug 18, 2026)
- S13 https://www.anthropic.com/news/updates-to-our-consumer-terms (Aug 28, 2025)
- S14 https://developers.google.com/terms (Nov 9, 2021)
- S15 https://cloud.google.com/terms (Mar 24, 2026)
- S16 https://developers.google.com/workspace/tools-safety
- S17 https://workspace.google.com/blog/product-announcements/enhancing-security-controls-for-google-drive-third-party-apps (May 31, 2019)
- S18 https://developers.google.com/identity/protocols/oauth2/resources/oob-migration
- S19 https://knowledge.workspace.google.com/admin/sync/transition-from-less-secure-apps-to-oauth (2026-10-26)
- S20 https://knowledge.workspace.google.com/admin/drive/storage-and-upload-limits-for-google-workspace (2026-10-26); https://forum.rclone.org/t/google-drive-service-account-changes-and-rclone/50136 (unverified)
- S21 https://developers.googleblog.com/en/usability-and-safety-updates-to-google-auth-platform/ (Apr 28, 2025)
- S22 https://developers.google.com/workspace/drive/api/guides/limits
- S23 https://appdefensealliance.dev/casa/casa-tiering (2026-06-27)
- S24 https://workspaceupdates.googleblog.com/2026/10/preview-edit-and-collaborate-on-Markdown-files-natively-across-Drive-and-Docs.html (Oct 5, 2026)
- S25 https://developers.google.com/workspace/drive/api/guides/ref-export-formats
- S26 https://developers.google.com/workspace/docs/api/limits
- S27 https://developers.google.com/workspace/sheets/api/limits
- S28 https://docs.cloud.google.com/pubsub/docs/publish-receive-messages-console
- S29 https://cloud.google.com/pubsub/pricing
- S30 https://cloud.google.com/run/pricing
- S31 https://developers.google.com/workspace/events/guides/events-drive
- S32 https://tacsecurity.com/google-casa-cloud-application-security-assessment/ ; https://www.switchlabs.dev/post/casa-tier-2-tier-3-security-review-providers-pricing-and-the-cheapest-option (unverified)
- S33 https://developers.google.com/workspace/marketplace/overview
- S34 https://developers.google.com/workspace/preview
- S35 https://developers.google.com/workspace/events/guides
