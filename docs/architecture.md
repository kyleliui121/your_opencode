# OpenCode Studio — Architecture & Technology Proposal

**Status:** Draft for discussion  
**Target:** Windows x64 first, cross-platform architecture  
**Product direction:** Conversation-first Agent desktop workspace; not an IDE

## 1. Product definition

OpenCode Studio is an independent desktop client for the OpenCode Agent runtime. Its primary interface is a conversation workspace. Users select a project, talk to an Agent, inspect its tool activity, review changes, approve actions, and continue work.

The product must not become a second IDE or a second Agent implementation.

### Core principles

1. **OpenCode remains the source of truth.** Do not create a parallel database for OpenCode sessions, provider credentials, or runtime configuration.
2. **Connect to the existing shared OpenCode service by default.** Do not start a private/standalone server unless the user explicitly chooses that mode.
3. **Use documented OpenCode APIs and SDKs.** Avoid direct reads/writes to OpenCode's SQLite database.
4. **Share state rather than copy state.** Official Desktop, CLI/TUI, and OpenCode Studio should see the same projects, sessions, credentials, and runtime state when they connect to the same local shared service.
5. **Keep app-specific state separate.** UI preferences, window layout, pinned projects, and local display settings belong to Studio, not OpenCode.
6. **Make compatibility explicit.** Detect the installed OpenCode version and capabilities; isolate experimental API usage behind an adapter.

## 2. Proposed technology stack

| Layer | Proposal | Responsibility |
|---|---|---|
| Desktop shell | Tauri 2 | Native window, installer, OS integration |
| UI | React + TypeScript + Vite | Conversation-first desktop interface |
| Styling | Tailwind CSS + Radix UI primitives | Consistent visual system and accessible controls |
| Icons | Lucide | Consistent iconography |
| Server-state cache | TanStack Query | API data fetching, cache invalidation, refresh |
| UI state | Zustand | Navigation, panel visibility, composer state, window layout |
| OpenCode API | `@opencode/client` | Typed client for the OpenCode HTTP API |
| OpenCode service discovery | Official `@opencode/client/service` where available | Discover/ensure the local shared service |
| Native layer | Rust | Process lifecycle, filesystem, OS keychain, notifications, deep links |
| Local app storage | SQLite or Tauri Store | Studio-only preferences and non-authoritative cache |
| Markdown | react-markdown + remark-gfm | Message rendering |
| Code highlighting | Shiki | Code blocks in Agent messages |
| Motion | Motion for React, used sparingly | Subtle transitions |
| Package manager | pnpm | Workspace and dependency management |
| Tests | Vitest + Playwright; Tauri integration tests for native boundaries | UI, adapter, and desktop verification |

### Important integration decision: service discovery

The official `@opencode/client/service` helper exposes `Service.discover()`, `Service.ensure()`, `Service.stop()`, and `Service.headers()`. The documentation describes this helper as a Node-side API; the normal `@opencode/client` entrypoint is browser-compatible.

Tauri's frontend is a WebView, so the service helper cannot simply be imported into React. Validate one of these approaches in a technical spike before locking the implementation:

- **Preferred compatibility path:** a small bundled Node sidecar uses the official service helper and `@opencode/client`; Tauri/Rust supervises it. Keep the sidecar's interface narrow.
- **Alternative:** implement service discovery in Rust only if the required registration and authentication contract can be consumed through documented/stable interfaces. Do not rely on undocumented service-file internals.
- **Do not use as the default:** repeatedly spawning `opencode api` for every request. It is useful for diagnostics, but not a good primary transport for streaming conversations.

The service bridge must bind only to loopback or use Tauri IPC/stdio. Never expose a local unauthenticated bridge to the LAN.

## 3. OpenCode shared-service integration

### 3.1 Default connection model

OpenCode's current local architecture uses a shared background server. Local clients normally discover or start this shared service. The service owns sessions, plugins, permissions, integrations, and other application state.

Studio should follow this sequence:

1. Detect the installed OpenCode CLI and read its version.
2. Discover the healthy registered shared service.
3. If it exists, connect to that exact service.
4. If it does not exist, use the official service lifecycle mechanism to ensure/start the shared service.
5. Verify the endpoint with the server info/health API.
6. Construct the typed OpenCode client with the service endpoint and its authentication headers.
7. Subscribe to events and load the initial snapshot of projects, sessions, providers, agents, MCP servers, and configuration.

Do not launch `opencode --standalone` as the default. A private server creates a separate runtime context and undermines the shared-state requirement.

### 3.2 Shared data and access strategy

| Data | OpenCode source of truth | Studio access strategy |
|---|---|---|
| Projects | Shared server project store | Project API |
| Sessions | Shared server session store | Session list/get/create APIs |
| Messages and tool parts | Shared server session history | Session message/context APIs |
| Session status | Shared server runtime | Session status and event stream |
| Provider/model availability | Shared server | Provider/model APIs |
| Global and project config | OpenCode config layers | Config API; show source and precedence |
| Agents | Loaded OpenCode configuration/runtime | Agent API |
| MCP servers | Loaded config/runtime | MCP API and status events |
| Skills and commands | Loaded OpenCode configuration/runtime | Skill/command APIs where supported |
| Credentials | Shared server credential store | Reuse server authentication; never read the DB directly |
| Permissions/forms | Shared server runtime | Permission/form APIs and events |
| Project files | Actual project filesystem | Native filesystem access, only when the user opens/selects the project |

### 3.3 Configuration layering

OpenCode supports multiple configuration sources and merges them by precedence. Sources can include remote organizational configuration, global configuration, custom configuration, project configuration, `.opencode` directories, environment-provided configuration, and managed configuration.

Studio must not parse only one global JSON file and present it as the effective configuration.

Use `GET /api/config` to inspect configuration documents and discovery sources for a location. Display the source and precedence for each relevant setting where possible.

For writes:

- Prefer official API operations when supported by the installed server version.
- The current global config patch route is marked experimental; guard it by version/capability checks.
- Project config editing must preserve JSONC comments and formatting. Do not rewrite the whole file through a lossy JSON serializer.
- For settings not safely editable through a stable API, provide an "Open source config" action rather than silently maintaining a shadow copy.
- After a persistent config change, refresh/reload the affected location using supported OpenCode behavior and re-query the effective configuration.

### 3.4 Credentials and provider accounts

Credentials are owned by the OpenCode server. Current OpenCode documentation states that saved API keys and OAuth tokens are stored in the server's SQLite database, and advises against editing the database directly.

Rules:

- Do not copy credentials into Studio's own database.
- Do not inspect or modify `opencode.db` or legacy `auth.json` to manage authentication.
- Do not call credential-list APIs that return secret values merely to populate a settings screen.
- Use provider/integration status APIs for display.
- Use official authentication flows/CLI commands for adding, switching, or removing accounts.
- Studio's own secrets, if any are ever needed, must go in the OS credential store.

### 3.5 Session interoperability

Sessions and messages must be read and written through the shared server APIs.

Required behavior:

- Show sessions created by official Desktop, CLI/TUI, or Studio.
- Continue an existing session by its original session ID and project/location.
- Show messages, tool calls, permission requests, and diffs from the server's session history.
- Sessions created by Studio must appear in other clients connected to the same shared service.
- Handle busy/conflict responses rather than submitting duplicate prompts.
- Do not create a parallel Studio-only transcript as the authoritative history.

Studio may keep a disposable read cache for fast rendering, but it must be reconstructible from OpenCode APIs and must never become the authoritative transcript.

### 3.6 Live synchronization and recovery

Subscribe to the OpenCode event stream for events such as:

- `session.created`, session metadata/status/execution/message/tool events
- `config.updated`
- `provider.updated`, `model.updated`, `agent.updated`
- `mcp.status.changed`
- `skill.updated`
- `permission.asked`, `permission.replied`
- `project.updated`

Important: the documented event stream is live-only. It does not replay missed events and does not automatically reconnect. Studio must implement:

1. Reconnect with bounded backoff.
2. Detect the server-connected marker and server identity/version changes.
3. Re-fetch the affected resource after reconnect.
4. On uncertain event loss, perform a full lightweight resync: projects, session list/status, active session messages, providers/models, agents, MCP status, and effective config.
5. Buffer events quickly; do not block the event consumer on expensive UI rendering.

Events are invalidation signals, not the sole source of truth. After receiving an event, fetch the canonical resource when needed.

## 4. Local state ownership

### OpenCode-owned state — never duplicate as authoritative

- Provider credentials and OAuth tokens
- OpenCode configuration
- Projects and session identity
- Session messages and tool-call history
- Agent runtime state
- Permission decisions
- MCP connection state

### Studio-owned state

- Window size and position
- Theme and visual preferences
- Sidebar/panel visibility and widths
- Recently opened project shortcuts (references only)
- Pinned conversations (references only)
- Composer draft text (optional)
- Local UI cache and cache version
- App-level accessibility preferences

Studio's local database must store OpenCode IDs and references, not shadow copies of OpenCode records.

## 5. Proposed module boundaries

```text
apps/
  desktop/
    src/
      app/                  # App shell and routing
      features/
        workspace/          # Project selection and recent workspaces
        conversations/      # Session list, transcript, composer
        activity/           # Tool calls, execution timeline, status
        approvals/           # Permission and form requests
        results/             # Diff summaries, artifacts, attachments
        settings/            # Studio preferences + OpenCode settings view
      services/
        opencode/
          client.ts          # Typed client construction
          service-bridge.ts  # Connection lifecycle abstraction
          adapter.ts         # Stable app-facing interface
          events.ts          # Event subscription and resync
          capabilities.ts    # Version/capability detection
      stores/                # UI-only Zustand stores
      styles/
    src-tauri/
      src/
        process/             # Sidecar/service process supervision
        filesystem/          # Native file access
        credentials/         # OS credential store
        notifications/
        ipc/

packages/
  opencode-contracts/        # App-facing types and compatibility mapping
  ui/                        # Shared UI components

docs/
  architecture.md
  compatibility.md           # Add after service spike
  design-system.md           # Add after UI prototype
```

The exact sidecar boundary is provisional until the service-discovery spike is complete.

## 6. Version and compatibility policy

- Detect CLI/server versions at startup.
- Pin the OpenCode CLI and `@opencode/client` versions used for development and packaging.
- Keep all OpenCode-specific calls inside one adapter module.
- Check endpoint/capability availability before using experimental routes.
- Do not assume the latest API surface exists on every installed version.
- If the installed server is too old or incompatible, explain the detected version and offer a clear upgrade/compatibility path; do not silently create a separate database.
- Maintain a small compatibility matrix for tested OpenCode versions.

## 7. Security and data safety

- Connect to the local shared service only through loopback.
- Keep service authentication headers out of logs and renderer-visible diagnostics.
- Do not expose the bridge on the LAN.
- Never log prompts, file contents, API keys, OAuth tokens, or authorization headers by default.
- Treat project files and Agent tool outputs as untrusted content.
- Do not write directly to OpenCode's SQLite database.
- Back up config files before any Studio-initiated edit.
- Clearly distinguish Studio preferences from OpenCode settings.

## 8. V1 scope

### V1 must have

1. Discover and connect to the existing shared OpenCode service.
2. Show the current OpenCode version and connection state.
3. List projects and sessions from the shared service.
4. Open an existing session and render its message/tool history.
5. Create a session and send a prompt.
6. Stream text and tool activity.
7. Display and resolve permission requests.
8. Show provider/model availability and effective configuration.
9. Reflect session/config/provider changes made by other local OpenCode clients.
10. Reconnect and resynchronize after service restart or event-stream interruption.

### V1 should not have

- A custom Agent loop or model runtime.
- A second credential manager.
- A second authoritative session database.
- A full code editor or IDE workspace.
- A custom MCP execution engine.
- Cross-device database synchronization.
- A broad plugin marketplace.

## 9. Technical spike and acceptance criteria

Before building the full UI, validate these flows on Windows:

1. Start official OpenCode and create at least one project and session.
2. Launch Studio and connect to the already-running shared service without starting a private server.
3. Confirm Studio lists the same projects and sessions.
4. Send a prompt from Studio and confirm the same session is visible in official OpenCode.
5. Send a prompt from official OpenCode and confirm Studio observes it after event delivery/resync.
6. Change a provider/model setting in official OpenCode and confirm Studio refreshes its view.
7. Restart the shared service while Studio is open; confirm Studio reconnects and restores the correct state.
8. Verify no direct SQLite writes, no credential copying, and no duplicate private service.

**Acceptance condition:** Studio and official OpenCode clients connected to the same local shared service operate on the same projects, sessions, configuration, credentials, and runtime state.

## 10. Scope boundary: local sharing vs cross-device sync

This proposal guarantees interoperability among clients connected to the same local shared OpenCode service.

It does not automatically synchronize two independent computers. Cross-device access would require a remote OpenCode server or a separate, explicitly designed sync mechanism. We must not sync raw SQLite database files between machines.

## 11. Official references

- OpenCode shared background service: https://opencode.ai/docs/cli
- OpenCode client SDK: https://opencode.ai/docs/build/client
- OpenCode API reference: https://opencode.ai/docs/api
- OpenCode configuration: https://opencode.ai/docs/config
- OpenCode troubleshooting and service files: https://opencode.ai/docs/troubleshooting
- OpenCode provider authentication: https://opencode.ai/docs/cli/providers
- Tauri 2: https://tauri.app/
