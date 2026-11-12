# OpenCode Studio

A standalone desktop client for OpenCode, designed as a conversation-first AI Agent workspace.

> Product direction: a refined, minimal desktop workspace centered on conversations and Agent task execution — not an IDE.

## Product principles

- Reuse the user's existing OpenCode installation, shared background service, configuration, credentials, projects, and sessions.
- Treat OpenCode as the source of truth for Agent runtime and persistent Agent data.
- Build a clean desktop experience around conversations, tool activity, task results, and approvals.
- Avoid maintaining a parallel Agent runtime or duplicating OpenCode's database.
- Start with Windows x64; keep the architecture portable.

## Status

- [x] Product direction selected: conversation-first workspace.
- [x] Initial architecture and synchronization requirements drafted.
- [ ] Validate shared-service discovery and connection in a technical spike.
- [ ] Build high-fidelity UI prototype.
- [ ] Implement the first end-to-end conversation flow.

## Documentation

- [Architecture and technology proposal](docs/architecture.md)

## License

MIT (planned). Third-party dependencies and bundled components retain their own licenses.
