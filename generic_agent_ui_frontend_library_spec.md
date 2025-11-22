# Generic Agent UI Frontend Library — Specification

**Version:** 0.1

**Date:** 2025-11-21

**Author:** ChatGPT (spec template)

---

## 1. Purpose & Goals

Build a reusable, framework-agnostic (but React-friendly) frontend library that provides a standardized UI for interacting with AI agents over REST APIs. It should be:

- **Generic:** connect to any agent implementing a minimal REST contract.
- **Composable:** provide discrete UI components (chat window, action palette, tool outputs, memory inspector, execution trace, streaming/real-time updates, etc.).
- **Integrable:** easy to drop into new or existing apps with minimal wiring.
- **Spec-driven:** behavior driven by JSON/YAML UI + API schema; supports runtime configuration.
- **Secure & Observable:** pluggable auth, telemetry hooks, error handling.


## 2. High-level Architecture

- **Core UI Library (framework-agnostic API):** stable JS package exposing configuration-driven APIs, events, and minimal runtime helpers.
- **React Adapter Package:** production-ready React components and hooks wrapping the core primitives (primary deliverable).
- **Theming & Assets Module:** CSS variables / Tailwind-compatible tokens, icons.
- **Integrations / Connectors:** adapters for common auth (OAuth2, API Key), websockets, SSE, and file upload backends.
- **Spec/Schema:** canonical JSON Schema (or OpenAPI fragment) describing how to configure UI, map endpoints, and declare capabilities.
- **CLI / Dev Tools:** generator for scaffolding a minimal integration and JSON spec validation.


## 3. Design Principles

- **Contract-first:** UI behavior follows the spec; no hard-coded assumptions about agent internals.
- **Extensible:** plugin points for custom renderers, message parsers, and tooling.
- **Declarative:** prefer configs over imperative code when wiring new agents.
- **Accessibility-first:** keyboard navigation, ARIA roles, color contrast, screen reader compatibility.
- **Performance-conscious:** lazy load heavy modules (e.g., visualizers), streaming-friendly.


## 4. Minimal REST Contract (Agent => UI)

To be generic the agent REST API should expose a minimal set of endpoints the UI can rely on. The spec will map to real endpoints.

### 4.1 Required endpoints (recommended path names only - configurable in spec)

1. **POST /sessions** — create session.
   - Request: `{ "user_id": "...", "context": {...} }`
   - Response: `201 { "session_id": "...", "expires_at": "ISO8601" }`

2. **POST /sessions/{session_id}/messages** — send user message / action request.
   - Request: `{ "message": { "role": "user", "content": "..." }, "metadata": {...} }`
   - Supports: `stream=true` query param for streaming responses.
   - Response (non-stream): `200 { "message_id": "...", "status": "accepted" }`

3. **GET /sessions/{session_id}/messages?after={cursor}** — list messages or incremental poll.
   - Response: `{ "messages": [ { message object } ], "cursor": "..." }`

4. **DELETE /sessions/{session_id}** — optional: end session.

5. **GET /capabilities** — optional but recommended: agent declares supported features (tools, multimodal, actions, streaming).
   - Response sample: `{"supports": {"tools": true, "streaming": true, "attachments": ["image","audio"]}}`

6. **POST /sessions/{session_id}/files** — file upload for multimodal inputs (optional)

7. **POST /sessions/{session_id}/actions/{action_name}** — run named tool/action (optional; an alternative is embedding an action intent in messages).


### 4.2 Message Object Schema

```json
{
  "message_id": "string",
  "role": "system|assistant|user|tool",
  "content": {
    "type": "text|markdown|json|rich|binary",
    "text": "...",
    "data": { },
    "attachments": [{"id":"...","type":"image"}]
  },
  "created_at": "ISO8601",
  "meta": { }
}
```


## 5. UI Feature Set (Core Components)

Each component is standalone, configurable, themeable, and emits events for host apps.

### 5.1 Session Manager
- Create, restore, and list sessions.
- Hooks for session persistence (localStorage, cookies, remote store).
- Configurable session lifetime and rehydration strategy.

### 5.2 Chat Window / Conversation View
- Message list with virtualized rendering.
- Rich message rendering pipeline (text, markdown, code blocks, JSON viewer, custom renderers).
- Message grouping, timestamps, delivery/read states.
- Inline action buttons (suggested replies, tool triggers).
- Composer: supports text, attachments, voice-to-text (pluggable), templates, message drafts.
- Streaming UI support: partial assistant tokens appended in real-time.

### 5.3 Action Palette / Tooling
- Declarative list of tools/actions the agent supports (from /capabilities or spec).
- Each action maps to an action endpoint and has a schema for parameters (UI form generator).
- Support for synchronous & asynchronous actions, polling action status.

### 5.4 Tool Output Renderers
- Generic renderers for: tables, charts (lightweight), images, code execution outputs, attachments, JSON, and custom components registered by host app.

### 5.5 Memory & Context Inspector
- Show conversation context, memory slots, system prompts, and metadata.
- Allow read-only and controlled-edit modes (if permitted by agent) for debugging.

### 5.6 Trace & Debug Panel (Dev Mode)
- Show API requests/responses (raw), latency, streaming frames.
- Toggleable by feature flag.

### 5.7 Notifications & Alerts
- Inline and toasts for errors, auth prompts, rate limits.

### 5.8 Accessibility
- Keyboard shortcuts for focus, send, navigate messages, open action palette.
- Screen reader announcements for streaming messages and completed responses.


## 6. Configuration Schema (spec-driven)

A JSON/YAML configuration that declares:
- Agent endpoints (base URL, auth scheme, endpoints mapping)
- Capabilities (tools, streaming, file types)
- UI features to enable/disable
- Message parsers and renderer registry
- Theme tokens
- Telemetry endpoints

**Example minimal config**

```yaml
agent:
  base_url: https://example.com/api/v1
  auth:
    type: apiKey
    header_name: X-API-KEY
endpoints:
  create_session: /sessions
  send_message: /sessions/{session_id}/messages
  poll_messages: /sessions/{session_id}/messages
capabilities_endpoint: /capabilities
ui:
  enableStreaming: true
  features:
    memoryInspector: true
    tracePanel: false
renderers:
  - type: markdown
  - type: jsonViewer
  - type: chart

```


## 7. Integration Contracts (Public API)

Library exposes:

### 7.1 Initialization
`initAgentUI(config: AgentUIConfig): AgentUIInstance`

### 7.2 Imperative APIs
- `createSession(context) -> Promise<Session>`
- `sendMessage(sessionId, message, opts) -> Promise<Message>`
- `on(eventName, handler)` — subscribe to events (`message.received`, `stream.delta`, `session.expired`, `error`)
- `registerRenderer(name, RendererComponent)` — extend rendering

### 7.3 React Hooks (React adapter)
- `useAgent(config)` returns `{ createSession, sessions, sendMessage, registerRenderer }`
- `useSession(sessionId)` returns reactive session state (messages, status)

### 7.4 UI Components (React)
- `<AgentProvider config>` — context provider
- `<SessionList />` — session UI
- `<Conversation sessionId />` — main chat view
- `<Composer sessionId />` — input area
- `<ActionPalette sessionId />` — tools
- `<TracePanel />`, `<MemoryInspector />` (dev components)

All components accept props for customization and callback props for event hooks.


## 8. Events & Observability

Event model should be consistent and documented. Key events:
- `session.created` `{sessionId}`
- `message.sent` `{message}`
- `message.received` `{message}`
- `stream.delta` `{chunk, meta}`
- `action.invoked` `{action, params}`
- `error.occurred` `{error, context}`

Telemetry/metrics hooks allow host app to forward events to its analytics stack.


## 9. Authentication & Security

- Support API keys, Bearer tokens (OAuth), cookie-based sessions.
- Token refresh flow hooks for host app to implement.
- CSRF considerations if using cookies.
- Encryption of persisted local caches (optional) and opt-out.
- Sanitize and limit logs for PII.
- Provide content moderation hooks (for host to plug moderation services before rendering messages).


## 10. Streaming & Real-Time

Support three modes:
- **Server-Sent Events (SSE)** — simple streaming.
- **WebSockets** — bi-directional real-time (optional).
- **Chunked HTTP/Fetch streaming** — append partial tokens from an OpenAI-like stream.

The library should expose an abstract `StreamHandler` interface so host can plug desired transport.


## 11. Error Handling & Retry

- Backoff retry strategies for transient errors; configurable limits.
- Distinguish between permanent and transient errors via HTTP codes.
- Expose retry hooks to host app to show UI controls (retry button).


## 12. Theming & Styling

- CSS variables and a theme token system (colors, radii, spacing, fonts).
- Tailwind-compatible classes provided in React package (optional).
- Provide a dark/light default theme.


## 13. Accessibility (a11y)

- Use semantic HTML for message list and composer.
- Provide ARIA live regions for streaming assistant output.
- Keyboard-only navigation and focus management.
- Ensure sufficient contrast and allow user font-size scaling.


## 14. Testing & Quality

- Unit tests for core logic, API contract parsing, renderer pipeline.
- Integration tests simulating agent responses (including streaming and errors).
- E2E tests for main flows (create session, send message, receive streaming reply, run action).
- Accessibility tests (axe-core integration).


## 15. Developer Experience

- Clear TypeScript types and JSON Schema for config.
- Storybook for UI components and renderer playground.
- Example demo apps showing integrations with a mock agent and with popular backends.
- CLI to scaffold integration and validate config.


## 16. Versioning & Compatibility

- Semantic versioning.
- Stable core API contract; breaking changes only for major releases.
- Migration docs for each major release.


## 17. Example Interaction Flows

1. **Basic Chat**
   - Host app initializes `AgentProvider` with config.
   - User opens conversation — `createSession` invoked.
   - User types and hits send — `POST /sessions/{id}/messages` is called with `stream=true`.
   - Library displays streaming tokens via `stream.delta` until `message.received` completed and stored.

2. **Triggering a Tool**
   - Agent or UI shows a suggested action.
   - User opens Action Palette and fills parameters (form generated via action schema).
   - `POST /sessions/{id}/actions/{name}` called. Tool output returned as `message` with `role: tool` and rendered with tool renderer.


## 18. Extensibility Points

- Renderer registry: `registerRenderer(typeName, { matcher, component })`
- Transport plugins: `registerTransport(name, transportImpl)`
- UI plugins: slot APIs for adding toolbars, footers, contextual panels


## 19. Sample API Mappings & Config Snippets

(Provided in companion repository / storybook. Include OpenAPI fragment to auto-generate client stubs.)


## 20. Security & Privacy Checklist

- Minimal logs for sensitive fields (mask PII)
- GDPR/CCPA compatibility notes for storing transcripts
- Clear opt-in for recording conversations
- Token rotation and secret storage guidance


## 21. Roadmap (next features)

- Multi-agent routing and federated sessions
- Built-in analytics dashboard
- Offline-first support and queueing
- Rich visual toolkits (graphs, custom canvas)
- Native mobile components (React Native)


## 22. Deliverables & Milestones

- M1: Core library + React adapter, basic chat & composer, config schema, README, tests.
- M2: Action palette, tools schema-driven UI, streaming & transports, Storybook.
- M3: Theming, accessibility hardening, CLI, example integrations.
- M4: Production readiness: security audit, perf benchmarks, docs site.


## 23. Appendix

- **Config JSON Schema (link)** — add to repo
- **OpenAPI fragments for agent** — sample
- **Message rendering patterns & best-practices**

---

*End of spec (v0.1)*



