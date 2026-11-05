# Round 3 notes: Google OAuth verification, admin controls, tokens, drive.file, hosted MCP (checked 2026-11-05)

Scope: input for `google-drive-grill.md`. Stamp = the page's "Last updated" where visible; "none" = no stamp. (unverified) = third-party only. (inferred) = not stated verbatim on any official page.

## 1. OAuth app verification and scope classes

| Item | Finding | Source |
|---|---|---|
| Internal user type | "Restricted to your own Workspace domain. Verification is not required." Exempt from the unverified-app screen and the 100-user cap; "An organization administrator may need to approve the app." The project must be owned by the org and the consent screen set to Internal. | S1 (none), S2 (none), S3 (none) |
| Internal + restricted scopes (drive.readonly / drive) | No page says restricted scopes re-trigger verification for Internal apps; S1 lists "Internal use only" as a verification exemption; S4 adds "Restricted Google Workspace scopes may also need administrator configuration" (= admin API controls, §3). Conclusion: no verification, no CASA, admin side only (inferred). | S1, S3, S4 |
| External: unverified cap | "100 new users in total, after the app presents the unverified app screen"; lifetime cap, not resettable; applies to unverified apps requesting sensitive/restricted scopes. | S4 (none), S3 |
| External: Testing status | Up to 100 listed test users; "Authorizations by a test user will expire seven days from the time of consent", including refresh tokens; exception only if the scopes are openid/email/profile. A Workspace admin marking the app "Trusted" overrides the 7-day limit and the test-user cap. | S4, S3, S5 |
| Brand verification | Homepage on an owned domain describing the app and linking the privacy policy; privacy policy on the same domain, linked from the consent screen; domain ownership via Search Console; Google branding guidelines; current project contacts. | S2 |
| Sensitive scopes | Brand verification + use-case review, demo video, scope justification. No security assessment. | S2 |
| Restricted scopes | All of the above + "annual security assessment from a Google empanelled group of security assessors"; re-verification annually. Google's pages disagree on the trigger: the Drive scope page says the assessment is needed "If you store restricted scope data on servers (or transmit)"; S6 says all restricted-scope apps undergo the annual assessment. Levels are AL1/AL2 on Google/ADA pages (not "Tier 2"); the ADA "Tier 2" page says the self-scan is deprecated and a lab-verified scan is required; revalidation every year; LOA valid 12 months. Cost: not on any official page; TAC Security advertises about $540/app/year for Tier 2 (unverified). | S2, S6 (none), S7, S8 (2026-06-27), S9 (2024-11-07) |
| Scope classes | Non-sensitive: drive.file, drive.appdata, drive.install. Sensitive: documents, documents.readonly, spreadsheets, spreadsheets.readonly, drive.apps.readonly. Restricted: drive, drive.readonly, drive.metadata, drive.metadata.readonly, drive.activity, drive.activity.readonly, drive.meet.readonly, drive.scripts. "Only specific application types are permitted to use restricted scopes for Google Drive." | S7 (none), S10 (none), S11 (none) |

## 2. "AI-powered agent" OAuth client flag (release note 2026-10-12)

- Rule: "Starting November 11, 2026, Google Workspace MCP servers will reject requests from any app or integration" whose OAuth client is not marked agentic. Explicitly NOT affected: "Apps and integrations that are not AI agents, like those that only call Google Workspace REST APIs." So it applies only to Google-hosted Workspace MCP servers, not to Drive/Docs/Sheets REST calls.
- Flag: checkbox "Use this client for an AI-powered agent" in Google Cloud console > Google Auth Platform > Clients (set at client creation).
- Lock-in: "OAuth clients that are not marked as agentic by November 12, 2026 will remain non-agentic" and can't call the MCP servers; fix = create a new agentic client. Mixed-use clients: create a separate agentic client.
- Service accounts cannot be flagged ("service accounts aren't managed in Google Auth Platform > Clients"); their MCP access is governed by domain-wide delegation settings.
- Source: S12 (release notes, entry dated 2026-10-12), S13.

## 3. Customer-side Workspace admin controls and token revocation

| Item | Finding | Source |
|---|---|---|
| App access control | Admin console > Security > Access and data control > API controls > Manage third-party app access. Per app: Trusted ("access to all Google Workspace services (OAuth scopes), including restricted services"), Limited ("only unrestricted Google services"), Specific Google data (listed scopes only), Blocked. | S14 (2026-10-26) |
| Unconfigured third-party apps default | "Allow users to access any third-party apps" is the default; alternatives: basic-info-only sign-in, or block all until configured. | S14 |
| Internal apps | Not automatically trusted by rule; separate checkbox "Trust internal apps" ("To allow API access for all internal apps, check the Trust internal apps box"). Default state of the checkbox: not documented. An admin may still mark any app Trusted/Limited/Blocked, including internal ones. | S14, S3 |
| Allowlist an external app | Manage App Access > Configure new app > enter app name or OAuth client ID > Search > select > choose scope/access level > Finish. Trusted = treated as internal for that org (overrides the 7-day Testing limit and the 100-test-user cap). | S14, S3 |
| Blocking drive.file apps | Per-app "Blocked" is not scope-limited (admins "can always block any OAuth app"). When a service (Drive) is set to Restricted, an optional checkbox lets users still grant "non-high-risk scopes, such as access to user-selected files" (drive.file); drive.file is not in the Drive high-risk list. | S14, S3 |
| Per-user revocation | The admin Security page lists a user's 3-legged tokens per app; revoked per user per app; the app then cannot access that user until re-authorized. | S15 |
| Refresh-token expiry rules | A token stops working when: the user revokes it; unused for 6 months; "user changed their password and the token includes Gmail scopes"; the account exceeds the maximum number of live tokens; an admin sets a service to Restricted (`admin_policy_enforced`); a GCP session length is exceeded; External + Testing project: 7 days (except openid/email/profile only). Limit: 100 refresh tokens per Google Account per OAuth client ID, oldest silently invalidated; service accounts exempt. | S16 (none) |
| Password reset | The official admin page only says third-party mail apps "will stop syncing data after a password reset, until a new OAuth 2.0 token is granted" and all app passwords are revoked; the sign-in cookie reset ends sessions. Drive-scope tokens: no statement (consistent with the S16 "Gmail scopes" rule). | S17 (2026-10-26), S16 |
| Suspended / deleted user | Suspend page: Drive/Gmail "unavailable to the user"; silent on OAuth grants. Nudge Security: grants persist through suspension and are restored on reactivation (unverified). Deleted user: not documented; the account is gone, so tokens cannot be refreshed (inferred). | S18 (2026-10-26), S19 (unverified) |

## 4. drive.file semantics, service accounts, domain-wide delegation

- (a) Server read later: the Picker desktop guide says "Pass the OAuth 2.0 token to the Drive API to read and modify files in which the user previously granted access"; no official text says "without the user present", but nothing restricts it beyond the normal refresh-token rules in §3 (6-month inactivity, 100-token cap per client, 7-day Testing for External). drive.file "works with all Drive API REST Resources". (inferred from S20 (none), S7)
- (b) Another user's token: no official page addresses it. The scope page: "lets users choose which files they want to share with your app", "per-file access"; grants are expressed through the picking user's OAuth token. Conclusion: a picked file is not reachable via a different user's drive.file token unless that user also picks/opens it (inferred, S7, S20).
- (c) Service account: "A service account is a special kind of account used by an application"; it needs files shared to its email unless DWD/admin roles ("directly share individual files with the service account's email address"); IAM roles "don't grant access to Google Workspace assets". "Service accounts don't have storage quota and can't own files" and "must upload files and folders into shared drives"; "service accounts do not belong to your Google Workspace domain", so domain-wide sharing does not reach them. Shared Drive membership with a role is implied by these pages; explicit confirmation is third-party only (unverified). S21 (none), S22 (none), S23 (none).
- (d) DWD: super admin only; Security > Access and data control > API controls > Manage Domain Wide Delegation > Add new > client ID (service account or OAuth client) + comma-delimited scopes; "scopes should be appropriately narrow"; up to 24 h propagation; multi-party approval may apply. Neither the admin page nor the OAuth service-account page mentions app verification or a restricted-scope assessment for DWD (verification is a consent-screen / public-app concept). Scopes needed for background reads of hand-picked files via impersonation: drive.readonly (restricted) or drive; drive.file only covers files the impersonated user opened/created with the app. Note: the "Domain-wide installation ... app verification is required" wording on S1 is about Marketplace domain-install, not service-account DWD. S24 (2026-10-26), S25 (none), S21.

## 5. Google-hosted Workspace MCP servers and the Claude.ai connector

| Item | Finding | Source |
|---|---|---|
| Status 2026-11-05 | Drive MCP: developer preview since 2026-04-22; Docs and Sheets MCP: developer preview since 2026-07-13; the 2026-10-12 entry still says "available as part of the Developer Preview Program". No GA entry. Endpoints: drivemcp/docsmcp/sheetsmcp.googleapis.com/mcp/v1. | S12, S13, S26 |
| Scopes | Drive: drive.readonly + drive.file. Docs: + documents.readonly, documents. Sheets: + spreadsheets.readonly, spreadsheets. | S13 (none), S27 (none) |
| File allowlist | None documented; access "Inherit[s] the same permissions and data governance controls as the user". | S27 |
| Pricing | No official pricing statement found. May 2026 blog: tiering/quota model for "agent tools", no price. | S13, S28 (2026-05-01) |
| Preview terms (Phase 3 blocker) | Developer Preview requires enrolment (Workspace account + Cloud project); "features in Developer Preview may not be included in public applications prior to the General Availability announcement"; no end users outside your domain/company without Google permission. | S29 (none) |
| Claude as client | Yes: Claude.ai and Claude Desktop are documented (redirect URI https://claude.ai/api/mcp/auth_callback, agentic checkbox, requires Claude Enterprise/Pro/Max/Team). Also Antigravity and "Others" (Streamable HTTP + OAuth 2.0). | S13, S27 |
| Anthropic built-in Drive connector scopes | Neither support.claude.com nor claude.com/docs lists scope names. claude.com/docs: "Google builds and runs the connector", each user signs in with their own account; capabilities: search, recent files, metadata/permissions, read content, create files. support.claude.com adds share/move/trash/upload/create folders (approval by default). Hence not limited to drive.readonly (inferred). Scope names drive.readonly + drive.file come from a user bug report (unverified); the Claude Tag admin connector (a different product) documents drive.readonly default + drive.file. | S30 ("Updated this week"), S31 (none), S32 (none) |

## Sources

- S1 https://developers.google.com/identity/protocols/oauth2/production-readiness/restricted-scope-verification
- S2 https://support.google.com/cloud/answer/13464321
- S3 https://developers.google.com/identity/protocols/oauth2/production-readiness/overview
- S4 https://support.google.com/cloud/answer/15549945
- S5 https://support.google.com/cloud/answer/13464323
- S6 https://support.google.com/cloud/answer/13465431
- S7 https://developers.google.com/workspace/drive/api/guides/api-specific-auth
- S8 https://appdefensealliance.dev/casa/casa-tiering
- S9 https://appdefensealliance.dev/casa/tier-2/tier2-overview
- S10 https://developers.google.com/workspace/docs/api/auth
- S11 https://developers.google.com/workspace/sheets/api/scopes
- S12 https://developers.google.com/workspace/release-notes
- S13 https://developers.google.com/workspace/guides/configure-mcp-servers
- S14 https://knowledge.workspace.google.com/admin/apps/control-which-apps-access-google-workspace-data (ex support.google.com/a/answer/7281227)
- S15 https://knowledge.workspace.google.com/admin/apps/how-authorized-access-works
- S16 https://developers.google.com/identity/protocols/oauth2 (Refresh token expiration)
- S17 https://knowledge.workspace.google.com/admin/users/reset-a-users-password
- S18 https://knowledge.workspace.google.com/admin/users/suspend-a-user-temporarily
- S19 https://www.nudgesecurity.com/post/oauth-grants-the-hanging-chads-of-suspended-google-workspace-users (unverified)
- S20 https://developers.google.com/workspace/drive/picker/guides/desktop-mobile-picker
- S21 https://developers.google.com/workspace/guides/create-credentials
- S22 https://developers.google.com/workspace/drive/api/guides/about-shareddrives
- S23 https://developers.google.com/workspace/drive/api/guides/manage-shareddrives
- S24 https://knowledge.workspace.google.com/admin/apps/control-api-access-with-domain-wide-delegation (ex support.google.com/a/answer/162106)
- S25 https://developers.google.com/identity/protocols/oauth2/service-account
- S26 https://docs.cloud.google.com/mcp/supported-products
- S27 https://developers.google.com/workspace/drive/api/guides/configure-mcp-server
- S28 https://workspaceupdates.googleblog.com/2026/05/agent-tools-and-security-updates-for-workspace-developers.html
- S29 https://developers.google.com/workspace/preview
- S30 https://support.claude.com/en/articles/10166901-use-google-workspace-connectors
- S31 https://claude.com/docs/connectors/google/drive
- S32 https://claude.com/docs/claude-tag/admins/connections/google
- Vendor cost claim: https://tacsecurity.com/?p=18721 (unverified)
