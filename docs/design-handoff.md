# dcars: Handoff Document

**dcars** (DocCyblade's Actual Receipt Scanner) is an independent community project. It is not made by, affiliated with, or endorsed by the Actual Budget project.

Status: idea settled, ready for planning. Personal project, intended to be open source (MIT).

## One-line summary
An installable web app (PWA) that photographs a receipt, uses AI to extract the details, matches the merchant to an existing Actual Budget payee, shows the exact transaction for confirmation, then submits it to a self-hosted Actual Budget server.

## Workflow
1. Take a photo in-app (preferred) or pick an existing photo.
2. Send the image to Claude's vision API; get structured JSON back.
3. Look for an existing payee in Actual. Fuzzy match. If none, propose creating a new one based on the detected details.
4. Show a preview of exactly what will be submitted to Actual.
5. User picks the account (never auto-chosen), can edit any field, then confirms.
6. Only after confirmation, submit the transaction.

## Decisions already made
| Topic | Decision |
|---|---|
| Name | `dcars`, all lowercase, for the repo, app and service. Avoid names that suggest an official Actual Budget product. |
| Platform | PWA, no Xcode, no Apple Developer fee. Works on iPhone, iPad, macOS (Add to Home Screen / Dock). |
| Hosting | Runs on the same Linux server as Actual (for example a Turnkey Linux Node.js appliance), Node.js already installed. |
| URL | Served from its own subdomain, e.g. `https://dcars.example.com`, behind a reverse proxy with HTTPS. |
| Actual server | Self-hosted Actual server, e.g. `https://actual.example.com`. May be on a home network reached by VPN. |
| Actual API | Official Node library `@actual-app/api` (https://actualbudget.org/docs/api/reference). No REST API exists, so the backend wraps the library. |
| AI | Pluggable AI provider. The AI call sits behind a provider interface; dcars ships one built-in Claude provider (Claude API with vision). Anyone wanting a different AI writes their own provider module against that interface; no other providers are built in. Receipt images go to the cloud. Each deployment supplies its own API key (for Claude: create at console.anthropic.com, prepaid credit, low spend limit). |
| Extraction scope | Total, date, merchant, plus an AI-suggested category. No line items or splits for v1. |
| Account | User chooses the account each time. |
| Encryption | The app MUST support an optional budget encryption password (config option passed to `downloadBudget`). |
| Offline | Scans queue on the device (IndexedDB) and upload when the server is reachable. Sync on next app open (iOS PWAs have no background upload). |
| Budgets | Multiple budget files and/or Actual servers supported. |
| License | MIT. No secrets or deployment-specific details in the repo. |
| Dev tooling | Built with Claude Code using Matt Pocock's engineering skills and workflow. |

## Architecture (proposed)
- One Node/TypeScript service: serves the PWA, exposes a small HTTP API, holds all secrets.
- Secrets (AI provider API key, secret encryption key) live in environment variables or a git-ignored config file. Never on the phone.
- AI provider abstraction:
  - The core app never calls an AI vendor or imports a vendor SDK directly. It calls a provider interface, roughly `extractReceipt(image, context) -> ReceiptExtraction`, where `context` carries the budget's existing categories (the AI may only pick from these).
  - `ReceiptExtraction` is one shared, provider-neutral shape (merchant, date, total in cents, suggested category, confidence and warnings). Every provider's output is validated against this schema before use, so a bad provider response fails clearly instead of reaching Actual.
  - The provider is chosen by config (e.g. `AI_PROVIDER=claude`). Each provider owns its own settings (API key, model name) and its own prompt.
  - v1 ships only the `claude` provider. Adding another AI means writing a new provider module and registering it; the docs include a short "writing a provider" guide and the Claude provider serves as the reference implementation.
- API sketch:
  - `POST /scan` (image) returns extracted JSON and a matched-payee preview.
  - `GET /meta` returns accounts, payees, categories.
  - `POST /confirm` (edited transaction) submits to Actual.
- Amounts are integer cents (Actual's format). Dates are ISO.
- Auth: modelled on Actual's own approach (password, then a session token). Each installed device registers and gets its own token, so every install is identifiable in logs. Users have roles: regular user and admin.
- Server-side storage (SQLite is enough): users, devices, budgets, scan records, and an audit log.

## Suggested milestones
1. Service only, tested with curl against a COPY of the budget: receipt image in, preview out, confirm call submits. Build the AI provider interface and the Claude provider here, not a direct Claude call to refactor later.
2. Payee fuzzy matching, category suggestion, create-new-payee flow.
3. Phone-friendly UI: camera/photo picker, account picker, edit and confirm screen.
4. PWA install, offline queue, polish, README (including the "writing a provider" guide), open-source release.

## Safety rules for development
- Build and test only against a copy of the budget (export and import as a second file) until stable.
- Nothing is ever submitted without an explicit confirm step.
- Never commit API keys, passwords, or deployment-specific details (domains, hostnames, Sync IDs, server paths). Docs and examples use placeholders like `example.com`.

## Risks and things to plan for
- Messy merchant names ("WALMART #1234" vs "Walmart"): needs a normalisation plus fuzzy-match strategy and a threshold for "no match".
- Duplicate detection (same receipt scanned twice).
- Receipts with unreadable totals or multiple totals: surface low confidence to the user, never guess silently.
- Actual API version must match the server version; pin and document it.
- Cost control: low spend limit on the API key; one image per scan.
- Camera and offline features require HTTPS.

## Resolved decisions
1. **Auth:** password plus access token, like Actual. Track each installed device for logging.
2. **Duplicates:** warn when date, amount and payee match an existing transaction; user can override.
3. **Currency/tax:** single currency (USD), total only. No separate tax tracking.
4. **Categories:** the AI may only choose from the user's existing Actual categories.
5. **Receipt images:** kept on the server, together with a record of every submission and who submitted it. Admins can see all users' records; regular users see their own.
6. **Proxy:** documented for Nginx; any HTTPS reverse proxy works.
7. **Name:** `dcars`.
8. **AI provider:** abstracted behind a provider interface. Claude is the only built-in provider; other AIs are supported by writing a provider module.

## Scope decisions
- **Multi-user and multi-budget from the start.** Budgets can be separate budget files or separate Actual servers.
- **Setup order:** register the device first, then select a budget. Configuration is organised by budget file/server, then by device.
- **Submissions are recorded per budget, per user, per device.** Admins can see everyone's records.
- **Device approval:** one-time invite code generated by an admin. Entering it registers the device.
- **Image retention:** keep forever (no auto-delete). Storage location and size to be planned (a disk-usage view for admins would help).
- **Typical use case:** a household of two people sharing one budget across several devices, so devices must be individually identifiable.

## Identity, access and operations
- **Identity:** each person has their own login; devices attach to a person. Records show who submitted and from which device.
- **Budget access:** per-person access. An admin grants each person access to specific budgets, and one is that person's default.
- **Budget secrets:** an admin adds budgets inside the app (Actual server URL, server password, Sync ID, optional encryption password). Secrets are encrypted at rest on the server.
- **First admin:** created from the command line on the server (no open sign-up screen).
- **Backups:** the app relies on the host's own backups. Docs must say which paths (SQLite database, image folder) to back up and how to restore.

## Security decisions
- **Secret encryption key:** stored in an environment variable on the server, never in the repo or database.
- **Login password:** separate from the Actual password. Each person sets their own for this app.
- **Lost devices and users:** an admin can deactivate a device or a user immediately.

## Remaining open questions
None. The planner may surface new ones during /grill-with-docs.

## Asked of the planner
Suggested flow: /setup-matt-pocock-skills once, then /grill-with-docs (start from this document), /to-spec, /to-tickets, /implement, /code-review. Start with milestone 1. Deployment-specific values for the developer's own setup are in a separate private notes file that is not committed.
