# dcars v1.0 roadmap

v1.0 is delivered as five milestones. Each one is written as a spec in the shape the `/to-spec` skill expects, so it can be refined with `/grill-with-docs`, published with `/to-spec`, and sliced into tracer-bullet tickets with `/to-tickets`.

The milestones build on each other in order. Source of the decisions: [design-handoff.md](design-handoff.md).

| # | Milestone | Done when |
|---|---|---|
| 1 | Receipt to preview to submit (service only) | A receipt image posted to the service yields a preview, and a confirm call creates the transaction in a copy of a budget |
| 2 | Matching, categories and duplicates | Messy merchant names resolve to the right payee, and a repeat scan is flagged |
| 3 | Users, devices and budgets | Two people can each log in from their own devices and submit to a shared budget, with admin control |
| 4 | Phone-friendly UI | A person can scan, review, edit and confirm entirely from a phone browser |
| 5 | PWA, offline and release | The app installs, queues scans offline, and is documented for public release |

## Conventions for every milestone

- **Amounts** are integer cents (Actual's format), single currency (USD), total only. Dates are ISO.
- **Nothing is submitted** to Actual without an explicit confirm step.
- **Safety:** build and test only against a copy of a budget. No secrets or deployment-specific values in the repo; docs use placeholders such as `example.com`.
- **Proposed test seam:** the service's HTTP API, with a fake AI provider and a throwaway Actual budget. One seam for the whole product keeps tests about external behavior. Each spec should confirm or replace this when it is written.

---

## Milestone 1: Receipt to preview to submit (service only)

### Problem Statement

Entering receipts into Actual by hand is slow and error-prone. Before any UI or user management exists, there must be a working core that proves the full path: an image goes in, structured data comes out, the person sees exactly what will be sent, and only then is it submitted.

### Solution

A Node/TypeScript service with a small HTTP API, tested with curl against a copy of a budget. Posting a receipt image returns an extraction and a preview of the transaction. A separate confirm call submits the (possibly edited) transaction to Actual. The AI call sits behind a provider interface from day one, with one built-in Claude provider.

### User Stories

1. As a household member, I want to post a receipt photo and get back the merchant, date and total, so that I do not have to read and type them myself.
2. As a household member, I want to see the exact transaction that will be created, so that I can check it before anything reaches my budget.
3. As a household member, I want to choose the account myself on every submission, so that a transaction is never filed to the wrong account.
4. As a household member, I want to correct any field before confirming, so that a misread total or date does not get saved.
5. As a household member, I want nothing submitted until I confirm, so that a scan can never change my budget by accident.
6. As a household member, I want to be told when a total or date could not be read reliably, so that I do not trust a guess.
7. As a household member, I want to list my accounts, payees and categories, so that I can pick valid values.
8. As an administrator, I want to set the AI provider, its API key and the secret encryption key through environment variables, so that no secret lives in the repo or on a phone.
9. As an administrator, I want to supply the optional budget encryption password, so that an encrypted budget can be opened.
10. As a developer, I want a provider interface with a shared, validated extraction shape, so that a bad provider response fails clearly instead of reaching Actual.
11. As a developer, I want the AI to choose only from the budget's existing categories, so that suggestions are always valid.

### Implementation Decisions

- One service holds all secrets and wraps the official `@actual-app/api` library; there is no REST API from Actual to call.
- The core never imports a vendor AI SDK. It calls a provider interface that takes an image plus a context carrying the budget's categories, and returns a provider-neutral extraction: merchant, date, total in cents, suggested category, confidence and warnings.
- Every provider response is validated against the shared extraction shape before use.
- The provider is selected by configuration. The Claude provider owns its own API key, model name and prompt, and serves as the reference implementation.
- API sketch: a scan call (image in; extraction and preview out), a metadata call (accounts, payees, categories), and a confirm call (edited transaction in; submitted to Actual).
- One image per scan, to control cost.
- Development runs against a copy of the budget, with the `@actual-app/api` version matched to the Actual server version.

### Testing Decisions

- A good test exercises external behavior only: image in, preview out, confirm creates the expected transaction.
- Tested through the HTTP API seam, with a fake provider for deterministic runs and a throwaway budget for the Actual side.
- A small set of real or sample receipts is used manually against the Claude provider; these are not committed.

### Out of Scope

Login and users, payee fuzzy matching beyond a simple lookup, duplicate detection, any UI, offline behavior.

### Further Notes

The provider interface and Claude provider are built here, not as a direct Claude call to refactor later.

---

## Milestone 2: Matching, categories and duplicates

### Problem Statement

Merchant names on receipts are messy ("WALMART #1234" versus "Walmart"). Without good matching, dcars would create duplicate payees, file transactions to the wrong payee, or double-enter the same receipt.

### Solution

Normalise merchant names, fuzzy-match them against existing Actual payees with a threshold for "no match", propose a new payee when none fits, suggest a category from the existing set, and warn when a transaction looks like a repeat.

### User Stories

1. As a household member, I want a scanned merchant matched to my existing payee, so that my reports stay clean.
2. As a household member, I want store numbers and punctuation ignored when matching, so that "WALMART #1234" finds "Walmart".
3. As a household member, I want to be offered a new payee when nothing matches well, so that a wrong match is never forced.
4. As a household member, I want to see why a payee was matched and how confident the match is, so that I can overrule it.
5. As a household member, I want a category suggested from my own categories, so that I rarely have to pick one.
6. As a household member, I want a warning when the date, amount and payee match an existing transaction, so that I do not enter a receipt twice.
7. As a household member, I want to override a duplicate warning, so that two genuine identical purchases can both be saved.
8. As a household member, I want a new payee to be created only after I confirm, so that scanning never alters my payee list silently.

### Implementation Decisions

- Normalisation plus fuzzy matching, with a documented threshold below which the result is "no match".
- The preview includes the matched or proposed payee, the match confidence, and any duplicate warning.
- A new payee is created in Actual only as part of the confirm step.
- Duplicate rule: same date, amount and payee as an existing transaction triggers a warning that the user can override.
- Category suggestions come only from the budget's existing categories.

### Testing Decisions

- Tested through the same HTTP API seam, using a throwaway budget seeded with known payees and transactions.
- Matching quality is covered with a table of messy-name examples and their expected outcomes.

### Out of Scope

Line items and splits, tax tracking, multiple currencies, learning from past corrections.

### Further Notes

Matching threshold values are decided during grilling and recorded as an ADR if they are contentious.

---

## Milestone 3: Users, devices and budgets

### Problem Statement

A household of two shares one budget across several devices. Submissions need to be attributable to a person and a device, secrets must be stored safely, and a lost phone must be cut off immediately. Some households also use more than one budget or Actual server.

### Solution

Per-person accounts with their own passwords, devices registered by one-time admin invite codes and given their own tokens, admin-managed budgets with encrypted secrets, per-person budget access with a default, and an audit trail of every submission with its receipt image.

### User Stories

1. As the first administrator, I want to create my account from the command line on the server, so that there is no open sign-up screen.
2. As an administrator, I want to generate a one-time invite code, so that only approved devices can register.
3. As a household member, I want to register a device by entering an invite code, so that my phone is recognised.
4. As a household member, I want my own password for dcars, separate from the Actual password, so that sharing one never exposes the other.
5. As an administrator, I want to add a budget (server address, server password, sync identifier, optional encryption password) inside the app, so that I do not edit server files.
6. As an administrator, I want those budget secrets encrypted at rest with a key held outside the database, so that a copied database does not reveal them.
7. As an administrator, I want to grant each person access to specific budgets and choose a default, so that people only see what they should.
8. As a household member, I want my submissions recorded with the budget, my name and my device, so that it is clear who added what.
9. As an administrator, I want to see everyone's records while regular users see only their own, so that privacy is respected.
10. As an administrator, I want to deactivate a device or a user immediately, so that a lost phone cannot submit.
11. As an administrator, I want receipt images kept with each record, so that a transaction can be checked later.
12. As an administrator, I want to see how much disk space receipt images use, so that I can plan storage.
13. As an administrator, I want documentation of which paths to back up and how to restore, so that I can protect the data.

### Implementation Decisions

- Auth is modelled on Actual's: a password, then a session token. Each registered device gets its own token so every install is identifiable in logs.
- Roles: regular user and administrator. Devices attach to a person.
- Setup order: register the device first, then select a budget. Configuration is organised by budget, then by device.
- Server-side storage (SQLite is enough): users, devices, budgets, scan records and an audit log.
- The secret encryption key comes from an environment variable and is never stored in the repo or the database.
- Receipt images are kept indefinitely on the server; there is no auto-delete.
- Backups rely on the host; the documentation names the database and image folder.

### Testing Decisions

- Tested through the HTTP API seam: registration, login, access control by role and budget, deactivation, and audit records.
- Security-relevant behavior (a deactivated device is rejected, a regular user cannot read another user's records) gets explicit tests.

### Out of Scope

Self-service sign-up, password-reset by email, single sign-on, automatic image deletion.

### Further Notes

Typical deployment is two people, one shared budget and about three devices.

---

## Milestone 4: Phone-friendly UI

### Problem Statement

A curl-driven service cannot be used standing in a shop. People need a simple screen flow on their phones: take a picture, check the result, pick an account, fix anything wrong, confirm.

### Solution

A mobile-first web UI served by the same service: sign in, capture or pick a photo, review the extracted transaction and payee match, choose the account, edit fields, confirm, and see recent submissions.

### User Stories

1. As a household member, I want to take a photo inside the app, so that scanning is quick.
2. As a household member, I want to pick an existing photo instead, so that I can scan a receipt I already photographed.
3. As a household member, I want a review screen showing merchant, date, total, payee match, category and warnings, so that I can verify everything at once.
4. As a household member, I want to choose the account on that screen, with no account preselected for me, so that I choose deliberately.
5. As a household member, I want to edit any field and see the preview update, so that I submit what I intend.
6. As a household member, I want clear low-confidence and duplicate warnings, so that I notice problems.
7. As a household member, I want a clear success or failure result after confirming, so that I know whether it was saved.
8. As a household member, I want to see my recent submissions, so that I can check what I entered.
9. As an administrator, I want to manage users, devices, invite codes and budgets from the UI, so that I do not need the command line after first setup.

### Implementation Decisions

- The UI is served by the same service and calls its HTTP API; it holds no secrets.
- Mobile first, usable on iPhone, iPad and desktop browsers.
- Camera access requires HTTPS.

### Testing Decisions

- Browser-level checks of the main flow (scan, review, edit, confirm) against the fake provider and a throwaway budget. These sit on top of the API seam and stay few.

### Out of Scope

Installability, offline use, push notifications, native apps.

### Further Notes

Visual design is decided during grilling or a prototype.

---

## Milestone 5: PWA, offline and release

### Problem Statement

The app should behave like an installed app on each device, tolerate poor connectivity at the shop, and be documented well enough for others to deploy it.

### Solution

Make the UI an installable PWA, queue scans on the device while offline and upload them when the server is reachable, then finish documentation and publish the open-source release.

### User Stories

1. As a household member, I want to add dcars to my Home Screen or Dock, so that it opens like an app.
2. As a household member, I want a scan taken offline to be saved on my device, so that I do not lose it when there is no connection.
3. As a household member, I want queued scans uploaded the next time I open the app with a connection, so that nothing is forgotten.
4. As a household member, I want to see what is queued and what has been sent, so that I know where each receipt is.
5. As a household member, I want queued scans to still require my review and confirm, so that offline scanning never auto-submits.
6. As an operator, I want a deployment guide covering the reverse proxy (Nginx documented), environment variables, and backups, so that I can run dcars myself.
7. As a developer, I want a "writing a provider" guide using the Claude provider as the reference, so that I can add another AI.
8. As a maintainer, I want a tagged release with a changelog, so that users know what changed.

### Implementation Decisions

- Offline scans queue in device storage (IndexedDB). iOS PWAs have no background upload, so the queue syncs on next app open.
- Documentation includes the pinned `@actual-app/api` version and the server version it matches.
- Release follows the changesets flow in `CLAUDE.md`.

### Testing Decisions

- Offline queue behavior tested at the browser level: enqueue while offline, sync once online, nothing submitted before confirm.
- Manual verification of install on an iPhone or iPad and a Mac.

### Out of Scope

Background sync, additional built-in AI providers, line-item or split transactions, multiple currencies.

### Further Notes

Before real use: confirm host backups cover the database and image folder, try one restore, test several real receipts against a test budget, and add the real budget last.
