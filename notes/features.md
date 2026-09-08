# Under development features

List of Codex agent/backend features that are flagged as "under development" and are disabled by default but can be enabled via config.

## Template

```markdown
### feature_name

<Description of the feature, what it does in practice, if it appears to supersede existing features, etc. - 2-4 sentences>

**Usability:** <Describes if, or to what extent, the feature is usable in frontends (e.g. TUI, VS Code extension) - Important to know if a feature is functional in the backend but not usable in any frontend - 1 short sentence>
```

## Feature List

### image_resize_notice

When image preparation resizes a data-URL image, this adds a separate developer message describing each affected image's original and prepared dimensions. It applies to images in user messages and tool outputs such as `view_image`, and the notice is persisted in history/rollouts and included in the model request alongside the resized image; it augments rather than replaces the existing image-preparation behavior.

**Usability:** The notice is available to backend and app-server raw-response-item consumers, but the TUI does not render it as ordinary chat content.

### local_thread_store_compression

When enabled for the local thread store, a best-effort startup worker scans active and archived rollout files and replaces files that have been cold for at least seven days with zstd-compressed `.jsonl.zst` files, including rollouts used as shared-history ancestors. Current readers stream compressed rollouts transparently, while append/resume and some new-reference operations materialize them back to plain JSONL; the worker verifies the compressed copy and preserves file metadata before deleting the original.

**Usability:** This is transparent storage infrastructure with no frontend control or visible UI, although all frontends using the local thread store benefit from the reduced disk usage.

### apply_patch_streaming_events

While the model is still generating a freeform `apply_patch` call, this parses the streamed patch input and emits `PatchApplyUpdated` events containing the latest structured snapshot of the file changes before the patch is executed. Snapshots identify each parsed path and add, delete, or update change, including partial file contents or update diffs; after the first update, emissions are coalesced for about 500 ms and the latest pending snapshot is flushed when the call ends. It only covers the custom/freeform `apply_patch` tool path, not ordinary function calls or patches embedded in shell commands, and incomplete input produces no update until it becomes parseable.

**Usability:** App-server v2 exposes these as `item/fileChange/patchUpdated` notifications for clients to render, but the in-repo TUI currently ignores them and therefore shows no live patch preview.

### mcp_2026_07_28

Enabling this selects MCP 2026-07-28 compatibility: streamable HTTP clients prefer `server/discover` and modern request metadata, while falling back to legacy `initialize` when the server only supports older protocol versions. Modern tool calls and resource reads can complete over multiple rounds using opaque request state plus elicitation/input responses; stdio uses the mode only when the server opts in with `CODEX_MCP_PROTOCOL_VERSION=2026-07-28` and then uses bounded JSON-RPC line framing.

**Usability:** The TUI and app-server expose MCP elicitation request/response plumbing, so user-interactive modern tools and resources can pause and resume, while legacy servers remain usable through fallback.

### non_prefixed_mcp_tool_names

When enabled, MCP tools are exposed to the model without the legacy `mcp__` namespace prefix, so `mcp__rmcp__echo` becomes `rmcp__echo`; this applies to ordinary MCP calls and code-mode nested tools. A table configuration can list exact server names that omit the prefix while other servers retain it, and the normal sanitization, uniqueness, and hash-suffix handling still applies. Dispatch continues to use the original MCP server and tool names, while MCP hook payloads keep the stable `mcp__...` naming form.

**Usability:** The shared backend makes this usable from the TUI, app-server, and code mode through configuration, but there is no dedicated frontend toggle and hook matchers should use the prefixed form.

### artifact

In the current checkout, this flag is effectively a no-op: `Feature::Artifact` remains registered as under development and can be enabled by the feature parser, but no artifact tool, handler, runtime, protocol surface, or frontend consumer remains, and the config schema deliberately omits the key. Earlier commits used it to gate native presentation/spreadsheet tools and then a freeform JavaScript `artifacts` tool backed by `@oai/artifact-tool`; that implementation was removed, so enabling the flag does not expose those capabilities now. The separate `thread_artifacts` storage model and migration are not connected to this feature flag.

**Usability:** None in the current build - enabling it produces no user-visible behavior.

### background_paginated_rollout_migration

For the local thread store, this starts a background one-shot migration that discovers active and archived legacy `.jsonl` or `.jsonl.zst` rollouts, canonicalizes their old records and rollback boundaries, materializes their SQLite history projection, and atomically publishes the rollout in paginated-history format while preserving compression and file metadata. A persisted creation-order cursor with a 48-hour lookback avoids rescanning everything on every startup; malformed, empty, and ordinary failed rollouts are recorded as skips so they do not block newer threads, while busy rollouts are retried later. Writer and maintenance locks, staged files, and a per-thread journal make concurrent archive/compression moves and interrupted publication recoverable, and app-server runtime enablement starts the worker once without providing a rollback path.

**Usability:** The TUI and app-server benefit automatically from migrated history, but no migration progress or completion UI is exposed; app-server can enable it through its experimental feature-enablement request.

### omit_app_server_notification_media

When enabled for a conversation, app-server `item/started`, `item/completed`, and `rawResponseItem/completed` notifications have inline image and audio content removed before delivery. It also removes MCP result image/audio items and binary resource blobs and clears image-generation results, while retaining text, metadata, encrypted content, and local media references; the filter does not change model input, stored thread data, or unrelated notifications such as realtime audio events. The feature applies only to outgoing app-server notifications, so thread reads and other APIs can still return the original media.

**Usability:** App-server clients can use it to reduce notification payload size and media exposure, but the direct TUI event stream and its user-facing settings have no dedicated support.

### powershell_shell_version

When enabled, Codex runs the selected local PowerShell executable with `-NoProfile -NonInteractive` and `$PSVersionTable.PSVersion.ToString()`, then exposes only its major and minor version in the model's `<environment_context>` as `<shell_version>` (for example, `5.1` or `7.4`). It runs only for exactly one ready local PowerShell environment, uses a two-second timeout, caches results by executable path including failures, and does not change shell selection or execution. If a previously visible version becomes unavailable, the next context diff explicitly marks it as unavailable so the model does not keep stale information.

**Usability:** This is model-only context used by the TUI and app-server through shared core configuration, with no dedicated frontend display or control.

### bedrock_setup_wizard

When enabled, eligible first-run TUI onboarding adds an "Use Amazon Bedrock" sign-in option. The wizard discovers local AWS profiles and environment credentials, then can configure a profile or environment credentials, save AWS access keys, or save a Bedrock API key; successful setup selects the `amazon-bedrock` model provider and persists the relevant provider settings. It appears only for the embedded app server with an unauthenticated default OpenAI provider and API login allowed, and the underlying app-server Bedrock RPCs are not themselves gated by this feature.

**Usability:** Fully usable in the TUI's initial onboarding flow; there is no separate settings UI, while app-server clients can call the experimental Bedrock RPCs independently.

### psp

When enabled, the effective HTTP client factory adds the process-scoped `oai-chat-psp=true` cookie to HTTPS requests for allowlisted ChatGPT hosts, allowing first-party ChatGPT services to route the request through PSP without changing the endpoint URL. Cookie-aware ChatGPT clients use it for first-party requests such as model discovery, authentication, task, and connector APIs, while API-host and arbitrary/custom-host requests do not receive the cookie; the client keeps the existing outbound proxy policy and only combines the marker with permitted Cloudflare affinity cookies. It changes routing selection at the service boundary and does not change model behavior, request payloads, or MCP traffic.

**Usability:** TUI and app-server requests that use ChatGPT authentication inherit the routing automatically from configuration, with no dedicated frontend control or visible UI.

### chronicle

In the current checkout, `Feature::Chronicle` is only a registered under-development flag, with the legacy `[features].telepathy` key mapped to it. No production code checks the flag, so enabling it does not launch a Chronicle/Telepathy sidecar, capture screen context, or change model prompts. The memories subsystem can ingest and prune externally supplied `memories/extensions/chronicle/` resources through its generic extension mechanism, but that mechanism is driven by `memory_tool`, not this flag.

**Usability:** There is no direct Chronicle functionality in the TUI or app-server; only generic memory-extension handling is usable when another component supplies the files.

### mcp_oauth_refresh_coordination

This flag selects a `Coordinated` MCP OAuth refresh mode, but in the current checkout the RMCP client immediately reports that coordination is unavailable and continues with the legacy path. The underlying OAuth persistor already performs cross-process locking, authoritative credential rereads, serialized refresh/persist transactions, and adoption of credentials refreshed by another process independently of this flag, so enabling it adds none of those behaviors. Because the selected mode is included in MCP connection identity, changing the flag during runtime configuration refresh can force an MCP reconnect; leaving it unchanged does not.

**Usability:** TUI and app-server configuration can enable it, but the only observable effects are the warning and possible reconnect, not a distinct OAuth behavior.

### realtime_conversation

When enabled for a thread, this unlocks the experimental app-server `thread/realtime/*` API: clients can start and stop a live voice or text session, send audio, text, or speech input, list supported voices, and receive transcript, audio, SDP, item, error, and close notifications. Sessions can use WebSocket, WebRTC from a browser SDP offer, or attach to an existing realtime call; V1, V2, and V3 select legacy Bidi, Realtime Voice, or Frameless Bidi behavior, with optional bounded startup context, V3 initial items, Codex handoff modes, automatic agent responses, and durable realtime timeline items for paginated threads. The RPCs also require the app-server client's `experimentalApi` capability, and requests against a thread created without this feature fail with an invalid-request error; WebRTC does not support V2 and existing-call attachment accepts only a restricted set of options.

**Usability:** The backend and app-server protocol are functional for custom clients, including browser/WebRTC clients, while the in-repo TUI only routes or ignores the notifications and provides no realtime capture or control UI.

### request_permissions_tool

When enabled and a runtime environment is available, Codex exposes the model-invoked `request_permissions` tool, which asks the host/client for additional filesystem or network permissions for a selected environment; relative paths resolve against that environment's cwd. A client can grant a subset at turn or session scope, and the grant is intersected with the originating sandbox policy before being applied automatically to later shell-like commands and `apply_patch` calls, with optional turn-scoped strict auto-review. `AskForApproval::Never` or granular approval with `request_permissions = false` auto-denies without prompting, while interactive approval is presented through the TUI or app-server; this is separate from inline `with_additional_permissions` command approvals.

**Usability:** Fully usable in the core, TUI approval overlay, and app-server clients, including remote environments; it has no effect unless enabled and invoked by the model.

### code_mode

When enabled, turns whose model metadata does not specify a tool mode default to `CodeMode`: the model receives a freeform `exec` tool and a JSON `wait` tool, and raw JavaScript runs in a V8-based code-mode host with helpers such as `tools.<name>`, `ALL_TOOLS`, `text`, `image`, `store`/`load`, and `yield_control`. JavaScript can compose eligible Codex tools, whose calls still use normal dispatch, approvals, and sandboxing; an execution can yield a cell for later waiting or termination, while ordinary direct tools remain available alongside the code-mode tools. A model-provided `tool_mode` takes precedence over this flag, and an unavailable host makes ordinary Code Mode fall back to direct tools unless fail-closed behavior is configured.

**Usability:** Usable through the shared core in the TUI and app-server as a model-facing capability with no dedicated frontend UI; execution requires the code-mode host to be available.

### code_mode_interrupt

When enabled alongside Code Mode, interrupting a turn terminates every currently active code-mode cell in that session, including a cell that yielded to run in the background and cells waiting on nested tool calls. The termination happens only for `TurnAbortReason::Interrupted`, not budget-limited turns; it does not enable Code Mode itself, and it leaves the reusable code-mode session state intact for later turns. After interruption, waiting on the terminated cell reports it as missing, while nested tool calls finish as aborted.

**Usability:** The normal TUI interrupt and app-server turn-interrupt APIs trigger it through shared core logic, but there is no separate frontend control or visible status indicator.

### respect_system_proxy

When enabled, Codex's shared HTTP client factory resolves a route for each destination using the host's system proxy settings before falling back to proxy environment variables (`HTTPS_PROXY`, `HTTP_PROXY`, `ALL_PROXY`, and `NO_PROXY`) and then a direct connection. On Windows this consults the current user's WinHTTP/Internet settings, explicit PAC URLs, WPAD auto-detection, static proxies, and bypass rules; macOS uses SystemConfiguration/CFNetwork, while platforms without a system resolver use the environment fallback. Route-aware clients cache route-specific transports and manually re-resolve redirects so a URL is not sent through a client selected for a different destination; this covers Codex-owned HTTP traffic such as auth, model/API, MCP, catalog, plugin, and update requests, but does not alter shell subprocess networking, and project-local config cannot enable the flag.

**Usability:** TUI and app-server traffic inherits the setting from shared configuration, with no dedicated frontend control or visible UI.

### code_mode_only

When enabled, this automatically enables Code Mode and makes `CodeModeOnly` the default tool mode for models whose metadata does not specify one. The model sees the Code Mode `exec` and `wait` entrypoints, while tools that can run as nested Code Mode calls - including shell, patch, dynamic, and eligible MCP/app tools - are removed from the top-level tool list and exposed through JavaScript helpers such as `tools.<name>` and `ALL_TOOLS`; direct-only exceptions such as `request_user_input` remain available, and configured namespaces can be forced to stay direct or be excluded from Code Mode. An explicit model `tool_mode` overrides the feature, and unlike ordinary Code Mode, an unavailable host never falls back to direct tools: the Code Mode call remains advertised but fails closed.

**Usability:** Usable by TUI and app-server model turns through configuration, with no dedicated frontend control; models without Code Mode support receive a warning.

### retain_client_developer_messages

When enabled, developer-role response items supplied through client injection paths such as app-server `thread/inject_items` or application additional context are tagged with private `client_authored` history metadata and persisted in the rollout; the metadata is stripped before provider requests. During Remote Compaction V2, these tagged messages are retained in the replacement history even though ordinary developer messages are discarded, and during Token Budget context resets they are copied into the new window; retention is bounded by a 64,000-token budget and may truncate or drop older messages. The flag has no effect on ordinary developer instructions, legacy remote compaction, or local summarization, and messages injected while it is disabled are not retroactively tagged.

**Usability:** App-server clients can supply the retained context through raw item injection or application context, while the TUI has no dedicated control or visible indicator.

### code_mode_prewarm

When enabled, each newly created thread asynchronously opens its durable Code Mode session during startup if a Code Mode host is available, so the first `exec` call can reuse the established session and avoid its connection/setup latency; it does not execute JavaScript or change model-visible tools. It runs independently of the `code_mode` feature, but is skipped when the host provider is disabled or unavailable; startup failures are logged without failing thread creation, and shutdown does not wait indefinitely for a stalled prewarm. Local threads lazily start the host process, while app-server can prewarm a configured remote gRPC host; the host or transport may be shared across threads, but session state remains per thread.

**Usability:** TUI and app-server threads benefit transparently from lower first-use latency, with configuration as the only control and no visible status UI.

### concurrent_reasoning_summaries

Despite its name, the current implementation is a reasoning-summary delivery mode rather than a compaction optimization. For OpenAI Responses requests that actually request a supported reasoning summary, it adds `stream_options.reasoning_summary_delivery = "sequential_cutoff"` on both HTTP and WebSocket requests, including WebSocket prewarm requests; this lets summary generation overlap with reasoning and allows an unfinished trailing summary section to be cut off when reasoning finishes. The client suppresses incremental summary-delta and summary-part events, then emits each completed `reasoning_summary_text.done` section as one reasoning update, with section breaks between completed sections; non-OpenAI providers, requests without summaries, compaction, and ordinary model output are unchanged.

**Usability:** TUI and app-server users see completed reasoning sections through the normal event stream, but have no frontend control over this delivery mode.

### rollout_budget

When enabled with a positive `limit_tokens` and reminder thresholds, this enforces one weighted usage budget shared by a root thread and its sub-agents. Each completed response consumes provider-reported `codex_rollout_budget_units` when present; otherwise it consumes `output_tokens * sampling_token_weight` plus non-cached input tokens * `prefill_token_weight`, with both weights defaulting to 1.0. Before sampling, the model receives a developer `<rollout_budget>` message showing the remaining weighted tokens, initially and whenever configured thresholds are crossed; a new context window or rollback causes the current remainder to be restated. Once usage reaches the limit, the current or later turn fails with `SessionBudgetExceeded` (including compaction), and the response can overshoot because accounting occurs after a completed response rather than reserving a per-request amount; invalid provider units fail fatally without retry.

**Usability:** It is usable through shared core behavior in the TUI and app-server, which surface the standard error, but neither frontend provides a dedicated budget display or control.

### context_management

When enabled at thread startup, this is an eligibility-gated activation switch for Codex's native context and history management. With ChatGPT authentication on a Plus, Pro, or ProLite account, an OpenAI Codex backend route, and no API-key or alternate provider credentials, it enables the `token_budget` feature and forces its private history/notes extension: the model gets context-window metadata and `new_context`/remaining-context behavior, can use private `history` and `notes` tools to recover prior windows and preserve notes across resets, and requests backend history ingestion. It replaces the legacy `notes.thread_hint` MCP bridge when active; other providers, auth modes, and plans get no effect.

**Usability:** Usable through shared core behavior in the TUI and app-server when configured, but the history/notes tools and context metadata are model-facing and have no dedicated frontend UI.

### current_time_reminder

When enabled, Codex reads the current UTC time before an inference request is due and persists a developer item such as `<current_time_reminder>It is 2026-06-17 17:34:15 UTC.</current_time_reminder>` in the model context; the default interval is one second, a new context window always triggers a reminder, and `delivery_mode = "after_user_or_tool_output"` suppresses reminders during internal continuation requests until user or tool output occurs. `clock_source = "system"` uses the host clock, while `"external"` asks the app-server client through experimental `currentTime/read` (with a 10-second timeout and exactly one subscribed client); failures stop the turn before the model request. The feature also exposes the model-facing `clock.curr_time` tool, and `sleep_tool = true` can expose an input-interruptible `clock.sleep` tool using the same clock, with sleeps capped at 12 hours.

**Usability:** System-clock reminders and clock tools work through the TUI and app-server, while external time requires a custom app-server client and is explicitly unsupported by the TUI.

### runtime_metrics

When enabled, Codex adds an OpenTelemetry manual/delta reader so the current turn can take on-demand snapshots of tool calls, API calls, SSE events, WebSocket requests/events, turn TTFT/TTFM, and Responses API timing fields. The TUI resets the snapshot at turn start, collects deltas while streaming, and accumulates them into the final turn separator; WebSocket sessions also send `x-responsesapi-include-timing-metrics: true`, which makes server overhead, inference, TTFT, and TBT values available for timing log entries and the final summary. The flag does not itself enable an OTEL metrics exporter, and the built-in Statsig exporter intentionally suppresses some metric names that a custom OTLP exporter records.

**Usability:** The TUI shows collected metrics inline in turn separators and WebSocket timing entries; app-server/backend users get exporter telemetry but no dedicated runtime-metrics UI or API.

### deferred_executor

When enabled, selected execution environments may remain `starting` while a turn continues instead of blocking session setup until their exec-server connection and environment metadata are ready. The model sees the pending environment in `<environment_context>` and receives a `wait_for_environment` tool that waits by environment ID; until it is ready, tools tied to that environment are withheld, while already-ready environments remain usable. After a successful wait, the next model step refreshes shell/filesystem access, MCP tools, AGENTS.md, skills, plugins, permissions, guardian execution, and child-agent environment inheritance; startup failure is reported as an unavailable environment rather than granting execution.

**Usability:** The shared core supports this in the TUI and app-server, but the full workflow requires a client that selects remote or provisioned environments and a model that invokes the wait tool; app-server also exposes environment registration and status APIs.

### deferred_tool_world_state

When enabled, Codex adds a model-visible `<tools>` developer world-state fragment listing namespaces whose tools are deferred, using a bounded one-line description for each namespace. The namespace map is persisted in rollout world state, so unchanged turns and resumed threads do not repeat it; additions, removals, description changes, and the transition to no remaining namespaces are sent as compact updates, with a 4 KiB rendered-fragment cap. It does not change which tools are deferred or how `tool_search` loads them - instead, the `tool_search` description omits its redundant source list because the same namespaces are already advertised in world state; empty state is neither rendered nor persisted.

**Usability:** TUI and app-server users benefit through shared model context, but neither frontend renders the `<tools>` fragment as user-facing UI or provides a dedicated control.

### shell_snapshot_v2

When enabled with Unified Exec, Codex asks a Unix-capable exec-server to capture each eligible shell's login state once and cache it in executor memory instead of writing a snapshot file under `CODEX_HOME`. Later direct `bash`, `zsh`, or `sh` commands in the same thread/environment restore functions, aliases, shell options, and policy-filtered exported variables, with explicit command environment overrides winning; captures are sandboxed, capped at 512 KiB, time out after 10 seconds, and failed captures fall back to normal execution with bounded retries. The first turn can asynchronously prewarm local snapshots after hooks when the project is trusted and no network proxy or network policy is active; remote Unix executors use lazy capture, while non-Unix executors and PowerShell/Cmd are unsupported, and eligible Unified Exec sessions replace the legacy file-backed snapshot path.

**Usability:** This is transparent to TUI and app-server shell commands on Unix executors that advertise the capability, with no frontend UI; local Windows execution does not use it.

### shell_zsh_fork

When enabled on Unix with the stable shell tool and Unified Exec, Codex can route zsh shell execution through a bundled patched zsh whose `execve(2)` path is redirected through `codex-execve-wrapper`. Each external subprocess launched by that shell is then evaluated separately by Codex's exec policy/Guardian and can run in the current sandbox, be approved and escalated, or be denied; this gives compound commands and persistent terminals per-subprocess enforcement rather than making the outer shell invocation the only approval boundary. The mode is selected only for zsh sessions when both the patched zsh and execve-wrapper paths are available; non-Unix platforms use the normal direct runtime, preparation failures fall back to direct execution, and remote environments are explicitly unsupported by the zsh-fork execution path.

**Usability:** Usable transparently by the TUI and app-server for local Unix zsh sessions in builds that supply the patched zsh and execve wrapper, with normal shell approval UI but no dedicated frontend control.

### skip_host_skill_discovery

When enabled, session initialization skips the eager host-skill snapshot scan only when the installed extension registry has skill-invocation contributors and none of them declares that it needs host-owned skills. Plugin discovery still runs first, but Codex then avoids `HostSkillsService::snapshot_for_config` and its host/user/repo/plugin skill-root filesystem discovery and startup error collection; empty registries and contributors that do not explicitly opt out retain the legacy scan. The skills extension opts out only when its provider set has no `HostSkillProvider`, while the current stock app-server registers a host provider alongside executor and orchestrator providers, so enabling this flag does not change normal app-server/TUI behavior.

**Usability:** This is a startup-I/O optimization for custom or alternate hosts using non-host skill providers; the stock TUI and app-server currently see no practical effect and expose no dedicated UI.

### standalone_web_search

When enabled for a provider/turn that supports namespace tools and web search, Codex exposes a client-executed `web.run` namespace tool and suppresses the normal hosted Responses API `web_search` tool. Calls are sent directly by Codex to the provider's `alpha/search` endpoint with the active model, web-search mode/settings, the current tool-output token budget, and a bounded text-only conversation tail (the latest two user messages plus at most 1,000 tokens of intervening assistant text); the command surface includes text/image search, open/click/find, PDF screenshots, finance, weather, sports, and time. The endpoint's plaintext `output` is returned to the model as a function-call result, while structured `results` are emitted and persisted as normal web-search items for clients; cached/indexed/live modes map to the standalone endpoint's external-web-access setting, and `web_search = "disabled"` prevents registration. Models marked `use_responses_lite` select this path automatically even without the flag, while ordinary models require the feature; custom providers must explicitly advertise standalone-search support.

**Usability:** App-server clients receive ordinary web-search item notifications/history and the TUI can render the existing web-search events, but there is no dedicated frontend toggle for choosing standalone versus hosted search.

### step_model_switching

Enables live per-turn settings changes for an already running turn. Through the app-server `turn/settings/update` API, a client can change the active turn's model, reasoning effort, reasoning-summary preference, or service tier without changing the thread settings inherited by future turns. Codex publishes the update into the active `TurnContext` as a new immutable `ResolvedStepSettings` snapshot; each later model-sampling step captures whatever snapshot is current at that moment, while steps, tool calls, approvals, and other actions that already captured an older snapshot keep their original settings. This allows a single turn to issue one Responses request with model A and, after an interaction such as `request_user_input`, continue the same turn/session/turn-state with model B. Model-dependent request state is rebuilt for later captures, including model metadata, reasoning/tier values, tool descriptions/availability, token-budget guidance, persistent-mode instructions, telemetry, input-modality filtering, and model-switch developer instructions.

The update is deliberately turn-local: `thread/settings/update` still changes only future-turn defaults, and a live turn update does not modify those defaults. Updates target an exact currently running task/turn ID and return `targetUnavailable` if that task is absent, completed, cancelled, or changed while the update is being prepared. Model changes are also rejected when the destination would alter security-sensitive semantics that still depend on the originally admitted turn, such as required approval authority, prefix-rule handling, node-REPL restrictions, Guardian reviewer/classifier/policy behavior, or managed constraints. Reviewer-only live updates are allowed independently; the feature gate is required for model/effort/summary/tier changes.

**Usability:** Exposed through the experimental app-server `turn/settings/update` API; there is no corresponding stock TUI control for switching the model in the middle of one active turn.

### terminal_visualization_instructions

A TUI-only prompt experiment. When enabled, the TUI appends a fixed four-bullet developer-instruction block to the thread's developer instructions on thread start, resume, and fork. It tells the model that the surface is a terminal; when other formatting rules call for a visual, to include a compact ASCII diagram, tree, timeline, or table in the final answer; to prefer tables for exact mappings/comparisons, trees for hierarchy or one-to-many relationships, and diagrams/timelines for sequence, change, or state transfer; and to use ASCII characters only in those visuals. It does not add any rendering capability, terminal protocol, tool, or post-processing—the only runtime effect is changing the model prompt. Existing developer instructions are preserved and the terminal guidance is appended after them; if there are no existing developer instructions, the guidance becomes the developer instructions by itself.

The injection is implemented in the TUI's app-server request construction, so it applies to TUI thread start/resume/fork flows (including the same parameter builders used for embedded/remote operation) but not generically to app-server clients. With the flag disabled, those builders retain their existing developer-instruction behavior.

**Usability:** Directly usable by enabling `[features].terminal_visualization_instructions`; there is no special visualization UI because the experiment only influences model formatting.

### token_budget

Replaces Codex's normal context-window compaction path with an explicit window-budget workflow. When enabled and the active model has a resolved context window, Codex injects developer-visible context-window metadata containing the agent path plus stable first/current/previous window UUIDs, exposes `get_context_remaining` and model-only `new_context` tools, and can inject configurable/model-owned guidance and near-limit reminders. Remaining tokens are computed against the tighter of the model's effective full context window and the configured auto-compaction scope; `get_context_remaining` reports that base (unbuffered) remainder. `new_context` requests a rollover without summarizing history and does not reset environment state.

Most importantly, auto/manual compaction under this feature does not invoke local or server summarization. It runs the normal pre/post compact hooks and emits a normal ContextCompaction lifecycle item, but then installs a fresh context window and drops the previous window's ordinary user/assistant/tool conversation from active model context. Window identity is advanced and persisted in Responses metadata so the backend/client can distinguish windows. Client-authored developer messages are retained only when the separate `retain_client_developer_messages` feature is enabled.

The feature has optional fallback behavior for models that need time to save state before rollover. At the base auto-compact threshold, a configured `auto_compact_fallback_prompt` can be injected once and Codex temporarily extends the auto-compaction limit by `auto_compact_fallback_buffer_tokens`; the model can use that extra space for note-taking and explicitly call `new_context`. If it consumes the buffer instead, Codex rolls over automatically. An explicit `new_context` request skips this fallback phase. A configurable `reminder_threshold_tokens` can also inject a one-shot reminder whose template substitutes `{n_remaining}`.

When `use_history_notes_extension` is enabled and the session is using OpenAI through the Codex backend, Codex additionally exposes private model-only `history.*` tools for listing/searching/reading prior context-window items and `notes.*` tools for persistent note files, requests backend history ingestion, and injects an optional backend-provided thread hint into the context-window slot. This extension is off by default for a bare `token_budget = true` configuration. Model metadata can provide token-budget defaults (including whether the feature should auto-enable, thresholds, guidance, fallback prompt/buffer, and history/notes preference); explicit user token-budget settings override those model defaults. The related `context_management.experimental_mode` path can also enable TokenBudget plus history/notes automatically, but only for eligible ChatGPT Plus/Pro/ProLite sessions using the first-party Codex backend with no provider credential override.

**Usability:** Directly usable through `[features.token_budget]` configuration. The model gets `get_context_remaining` and `new_context` automatically; there is no dedicated TUI control required. The history/notes portion additionally depends on first-party Codex-backend authentication.

### transcript_v2

Currently a registered but unused feature flag. Its declaration says it is intended to enable an "interactive transcript composer and turn-selection UI", and it can be parsed from `[features].transcript_v2` / persisted by `codex features enable transcript_v2`, but there is no runtime consumer of `Feature::TranscriptV2` in the current code. The introducing commit (#40554, 2026-08-25) was explicitly registration-only: it added the enum/spec entry, generated config-schema field, a feature-resolution test, and CLI persistence coverage, without adding TUI/core/app-server behavior. Current code search still finds the enum only in the feature registry and its toggle test, so enabling it does not activate a different transcript, composer, turn picker, protocol, or rendering path.

**Usability:** No practical effect in this revision. It is a placeholder/gate for planned transcript-v2 work rather than an implemented experiment.

### unified_image_budget

Unifies Codex's image preprocessing and local context-cost accounting so every prompt image uses one model-sizing policy instead of the legacy `detail` split. When the feature is enabled **and** the active model either advertises `supports_image_detail_original` or uses Responses Lite, all inline data-URL images from user messages and tool outputs are prepared with the same limits: maximum dimension 6,000 px and at most 10,000 32×32 patches. Images are downscaled only enough to satisfy those two limits. Their effective detail is recorded as `original`, and Codex's context estimator charges the prepared image by its 32×32 patch count (capped at 10,000) rather than the legacy fixed high-detail estimate. This makes the preprocessing ceiling and local token accounting use the same patch budget.

Without the feature, `auto`/missing/`high` detail uses the legacy high-detail limits of 2,048 px and 2,500 patches, while explicit `original` uses the 6,000 px / 10,000-patch limits; `low` is rejected/replaced as unsupported. In unified mode the old detail hint no longer selects a policy: even legacy `high`, `original`, `auto`, or `low` inputs are handled under the same unified budget. Codex writes `detail: "original"` into prepared history items as a compatibility/accounting marker; Responses Lite strips image-detail fields from the actual outgoing request because that transport does not use them.

The visible tool contract changes accordingly. `view_image` stops advertising or returning a `detail` parameter/result field and always feeds the loaded image through the unified budget; for backward compatibility the handler still accepts old callers that send `detail: "high"` or `"original"`, but the hint is ignored for sizing. Code Mode similarly hides image-detail selection from its generated tool surface. Central preparation still applies to ordinary user images and tool-output images, so the same policy also feeds retained-history/compaction token estimates and Guardian image preparation where that model supports the unified mode. The separate `image_resize_notice` feature, if enabled, can report any resulting source/prepared dimension changes.

The feature deliberately stays inactive for a normal Responses model that neither supports original image detail nor uses Responses Lite; merely setting the flag then preserves the legacy detail-based behavior.

**Usability:** Directly usable with `[features].unified_image_budget = true`, but only effective for compatible models. There is no dedicated TUI control; its user-visible effect is higher/consistent image-resolution budgeting and removal of the model-facing `view_image.detail` choice.
