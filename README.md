# dcars

**DocCyblade's Actual Receipt Scanner.** Photograph a receipt, let AI read it, review the transaction, and send it to your self-hosted [Actual Budget](https://actualbudget.org) server.

> dcars is an independent community project. It is not made by, affiliated with, or endorsed by the Actual Budget project.

**Status:** pre-release, in planning. Nothing is installable yet. See the [v1.0 roadmap](docs/ROADMAP.md).

## How it works

1. Take a photo in the app, or pick an existing one.
2. The image is sent to an AI provider, which returns the merchant, date, total and a suggested category.
3. dcars looks for a matching Actual payee (fuzzy match) or proposes a new one.
4. You see exactly what will be submitted, choose the account, and edit any field.
5. Only after you confirm is the transaction sent to Actual.

Nothing is ever submitted without that confirm step.

## Planned features for v1.0

- Installable web app (PWA) for iPhone, iPad and desktop; no app store, no Xcode.
- Pluggable AI provider. Claude (vision API) is built in; other AIs are added by writing a provider module.
- Payee matching, category suggestion limited to your existing categories, duplicate warning.
- Multiple people, devices and budgets, with per-person login, admin roles and invite-code device registration.
- Optional Actual budget encryption password support.
- Offline queue: scans are stored on the device and uploaded when the server is reachable.

## Requirements

- A self-hosted Actual Budget server.
- Node.js 22 or newer (the minimum required by `@actual-app/api`).
- The `@actual-app/api` version must match your Actual server version; dcars will pin and document it.
- An API key for your chosen AI provider (for Claude, from console.anthropic.com). Receipt images are sent to that provider's cloud service.
- An HTTPS reverse proxy (for example Nginx). Camera access and offline features require HTTPS.

## Documentation

- [Roadmap for v1.0](docs/ROADMAP.md): milestones written as specs.
- [Design handoff](docs/design-handoff.md): the decisions and architecture the project started from.

## Development

dcars is built with Claude Code and [Matt Pocock's engineering skills](https://github.com/mattpocock/skills). The flow is `/grill-with-docs`, `/to-spec`, `/to-tickets`, `/implement`, then `/code-review`. Issues and specs live in GitHub Issues; see `docs/agents/` for the configuration.

Branching is trunk-based on `main`. Every change is a short-lived branch and a pull request, and user-facing changes include a [changeset](https://github.com/changesets/changesets) (`npx changeset`). See `CLAUDE.md` for the full rules.

Safety rules while developing: test only against a copy of your budget, and never commit API keys, passwords or deployment-specific details such as hostnames or Sync IDs. Docs and examples use placeholders like `example.com`.

## License

[MIT](LICENSE)
