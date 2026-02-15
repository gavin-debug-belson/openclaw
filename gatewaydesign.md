# OpenClaw Gateway Design & Operation Sequence

This document describes the architecture and sequence of operations performed by the OpenClaw Gateway (`src/gateway`).

## Table of Contents

1. [Overview](#overview)
2. [Main Entry Points](#main-entry-points)
3. [Architecture Components](#architecture-components)
4. [Initialization Sequence](#initialization-sequence)
5. [Request Handling Flow](#request-handling-flow)
6. [Key Subsystems](#key-subsystems)
7. [WebSocket Protocol](#websocket-protocol)
8. [HTTP Endpoints](#http-endpoints)
9. [State Management](#state-management)
10. [Plugin System](#plugin-system)
11. [Security & Authentication](#security--authentication)

## Overview

The OpenClaw Gateway is a **multi-protocol, event-driven server** that acts as the central coordination point for the OpenClaw system. It:

- Accepts connections via WebSocket (primary) and HTTP (REST APIs)
- Authenticates clients using multiple mechanisms
- Routes requests to 60+ built-in methods plus plugin-provided methods
- Manages state for chats, nodes, sessions, approvals, and cron jobs
- Broadcasts events to all connected clients in real-time
- Coordinates distributed agent execution across multiple nodes
- Integrates plugins for extensibility
- Hot-reloads configuration without disrupting active connections

## Main Entry Points

### `server.ts`

Minimal re-export file that exposes the public API:
- `startGatewayServer(port, options)` - Main function to start the gateway
- `truncateCloseReason()` - Utility for formatting close messages

### `server.impl.ts`

Core implementation (~800+ lines) containing the main `startGatewayServer()` function that orchestrates the entire gateway startup sequence. This is the heart of the gateway.

### `boot.ts`

Startup sequence handler that optionally runs `BOOT.md` automation instructions before clients connect. This allows automated setup tasks to run at gateway startup.

## Architecture Components

| Component | Primary File(s) | Responsibility |
|-----------|----------------|----------------|
| **Config Management** | `server-runtime-config.ts` | Resolves runtime configuration (auth, TLS, control UI, bind mode) |
| **WebSocket Server** | `server-ws-runtime.ts`, `server/ws-connection.ts` | Manages WS connections, message routing, auth handshakes |
| **HTTP Server** | `server-http.ts`, `server/http-listen.ts` | Handles REST endpoints (OpenAI, OpenResponses, Hooks, Tools) |
| **Request Routing** | `server-methods.ts`, `server-methods/*.ts` | Dispatches WebSocket/HTTP requests to handlers with authorization |
| **Chat Management** | `server-chat.ts` | Manages agent chat runs, streaming, event handling |
| **Node Registry** | `node-registry.ts` | Tracks connected nodes and remote agent capabilities |
| **Plugin System** | `server-plugins.ts` | Loads and manages plugins, extends gateway functionality |
| **Channel Manager** | `server-channels.ts` | Manages message channels (Slack, Discord, Teams, etc.) |
| **Cron Service** | `server-cron.ts` | Manages scheduled jobs and automation |
| **Discovery** | `server-discovery-runtime.ts` | Bonjour/mDNS and Tailscale discovery |
| **Health Monitoring** | `server/health-state.ts`, `server-maintenance.ts` | Tracks gateway & agent health status |
| **Execution Approvals** | `exec-approval-manager.ts` | Manages approval workflows for sensitive operations |
| **Session Management** | `session-utils.ts`, `sessions-resolve.ts` | Session state and persistence |
| **Authentication** | `auth.ts`, `auth-rate-limit.ts`, `device-auth.ts` | Multi-method authentication and rate limiting |

## Initialization Sequence

The gateway follows a precise initialization sequence in `startGatewayServer()`:

### Phase 1: Configuration Loading & Validation

1. **Read Config File** (`readConfigFileSnapshot()`)
   - Load configuration from `~/.openclaw/config.json`
   - Validate against schema

2. **Legacy Migration** (`migrateLegacyConfig()`)
   - Auto-migrate old config entries if detected
   - Write updated config back to disk
   - Log migration changes

3. **Config Validation**
   - Check for schema violations
   - Throw error with helpful message if invalid
   - Suggest running `openclaw doctor` for repairs

4. **Plugin Auto-Enable** (`applyPluginAutoEnable()`)
   - Detect environment variables for plugins
   - Auto-enable plugins based on env config
   - Write updated config if changes made

5. **Load Final Config** (`loadConfig()`)
   - Load fully validated and migrated config
   - This becomes `cfgAtStart` used throughout initialization

### Phase 2: Core System Setup

6. **Diagnostics** (`isDiagnosticsEnabled()`, `startDiagnosticHeartbeat()`)
   - Start diagnostic heartbeat if enabled
   - Used for monitoring and telemetry

7. **Restart Policy** (`setGatewaySigusr1RestartPolicy()`)
   - Configure SIGUSR1 signal handling for graceful restarts
   - Set deferral checks for active operations

8. **Subagent Registry** (`initSubagentRegistry()`)
   - Initialize registry for tracking sub-agents

9. **Workspace Resolution**
   - Resolve default agent ID and workspace directory
   - Used for plugin loading and session storage

### Phase 3: Plugin Loading

10. **Load Plugins** (`loadGatewayPlugins()`)
    - Scan and load plugins from filesystem
    - Merge plugin-provided gateway handlers
    - Register plugin-provided tools, hooks, channels
    - Build combined gateway methods list

11. **Channel Plugin Setup**
    - Create channel logs for each plugin
    - Build runtime environments for channels
    - Merge channel methods with gateway methods

### Phase 4: Runtime Configuration

12. **Resolve Runtime Config** (`resolveGatewayRuntimeConfig()`)
    - Determine bind address (loopback/LAN/tailnet)
    - Resolve auth configuration
    - Determine enabled features (Control UI, OpenAI endpoints, etc.)
    - Setup TLS if configured

13. **Auth Rate Limiter** (`createAuthRateLimiter()`)
    - Create rate limiter if configured in config
    - Protects against brute-force attacks

14. **Control UI Assets** (`ensureControlUiAssetsBuilt()`)
    - Ensure web UI assets are built
    - Resolve Control UI root directory

15. **Wizard Session Tracker** (`createWizardSessionTracker()`)
    - Create tracker for onboarding wizard sessions

16. **TLS Setup** (`loadGatewayTlsRuntime()`)
    - Load TLS certificates if enabled
    - Validate certificate configuration

### Phase 5: Gateway Runtime State

17. **Create Runtime State** (`createGatewayRuntimeState()`)
    - Create HTTP/HTTPS servers
    - Create WebSocket server (WSS)
    - Setup Canvas host (if enabled)
    - Create broadcast mechanism
    - Initialize chat run tracking
    - Create client connection pool
    - Setup deduplication for chat runs
    - Initialize agent run sequence tracking

### Phase 6: Core Services Initialization

18. **Node Registry** (`new NodeRegistry()`)
    - Create registry for connected agent nodes
    - Setup node presence tracking

19. **Node Subscriptions** (`createNodeSubscriptionManager()`)
    - Create subscription manager for node events
    - Setup event routing to sessions

20. **Channel Manager** (`createChannelManager()`)
    - Create manager for messaging channels
    - Prepare for channel startup

21. **Cron Service** (`buildGatewayCronService()`)
    - Build cron service with configuration
    - Prepare scheduled jobs

22. **Execution Approval System**
    - Create execution approval manager
    - Setup approval handlers
    - Create approval forwarder for remote nodes

### Phase 7: Event System Setup

23. **Agent Event Handlers** (`onAgentEvent()`)
    - Subscribe to agent lifecycle events
    - Setup streaming chat event handling
    - Configure broadcast logic

24. **Heartbeat Event Handlers** (`onHeartbeatEvent()`)
    - Subscribe to heartbeat events
    - Setup node presence updates

25. **Skills Registry** (`setSkillsRemoteRegistry()`)
    - Setup remote skills cache
    - Configure skills refresh mechanism

### Phase 8: Discovery Services

26. **Gateway Discovery** (`startGatewayDiscovery()`)
    - Start Bonjour/mDNS discovery (unless minimal test mode)
    - Advertise gateway on local network
    - Store stop function for cleanup

### Phase 9: Maintenance & Monitoring

27. **Maintenance Timers** (`startGatewayMaintenanceTimers()`)
    - Start health check broadcasts (~every 5 seconds)
    - Start deduplication cleanup (stale chat runs)
    - Setup tick broadcasts
    - Configure lane concurrency limits

28. **Heartbeat Runner** (`startHeartbeatRunner()`)
    - Start heartbeat runner for health monitoring
    - Configure heartbeat intervals

### Phase 10: Configuration Hot-Reload

29. **Config Reloader** (`startGatewayConfigReloader()`)
    - Start file watcher on config.json
    - Setup hot-reload handlers
    - Configure reload for cron, hooks, heartbeat config

### Phase 11: WebSocket Handler Setup

30. **Attach WebSocket Handlers** (`attachGatewayWsHandlers()`)
    - Setup connection authentication
    - Configure message routing
    - Inject request context
    - Setup broadcast mechanism

### Phase 12: HTTP Server Listening

31. **Start HTTP Server(s)** (`listenGatewayHttpServer()`)
    - Bind HTTP/HTTPS servers to configured addresses
    - Start listening for connections
    - Log bind addresses and ports

### Phase 13: Sidecar Services

32. **Start Sidecars** (`startGatewaySidecars()`)
    - **Browser Control Server** - Start if enabled
    - **Gmail Watcher** - Start if configured
    - **Internal Hooks** - Load hook handlers
    - **Channel Startup** - Launch configured channels
    - **Plugin Services** - Start plugin background services
    - **Memory Backend** - Initialize memory backend

### Phase 14: Tailscale Exposure

33. **Tailscale Exposure** (`startGatewayTailscaleExposure()`)
    - Expose gateway to Tailscale network if configured
    - Configure funnel or serve mode

### Phase 15: Post-Startup Tasks

34. **Gateway Start Hook** (`gateway_start`)
    - Execute plugin hooks for gateway startup

35. **Update Check** (`scheduleGatewayUpdateCheck()`)
    - Schedule update check for CLI/gateway

36. **Boot Sequence** (`runBootOnce()`)
    - Run BOOT.md instructions if present
    - Execute startup automation

37. **Skills Cache Prime** (`primeRemoteSkillsCache()`)
    - Prime remote skills cache for connected nodes

38. **Log Startup** (`logGatewayStartup()`)
    - Log comprehensive startup information
    - Display bind addresses, auth mode, enabled features

39. **Return Server Handle**
    - Return `GatewayServer` object with `close()` method
    - Gateway is now ready to accept connections

## Request Handling Flow

### WebSocket Request Flow

```
┌─────────────────────────────────────────────────────────────┐
│ Client sends RequestFrame over WebSocket                    │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ ws-connection.ts: attachGatewayWsConnectionHandler()        │
├─────────────────────────────────────────────────────────────┤
│ • Connection upgrade from HTTP                              │
│ • WebSocket handshake                                       │
│ • Assign unique connection ID                               │
│ • Setup ping/pong for keepalive                            │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ ws-connection/message-handler.ts: Message Processing        │
├─────────────────────────────────────────────────────────────┤
│ 1. Buffer incoming message chunks                           │
│ 2. Check against max payload size (default 100MB)          │
│ 3. Parse complete message as JSON                           │
│ 4. Validate against protocol schema                         │
│ 5. Extract frame type (request/response/event)             │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Authentication & Authorization                               │
├─────────────────────────────────────────────────────────────┤
│ • Extract auth credentials from frame:                      │
│   - Bearer token                                             │
│   - Password (hashed)                                        │
│   - Device token (public key signature)                     │
│   - Tailscale user identity                                 │
│ • Auth rate limiter check (if configured)                  │
│ • Verify credentials against resolvedAuth config           │
│ • Check client certificate for device auth (optional)      │
│ • Determine client role (admin/read/write/pairing)        │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Build GatewayWsClient Object                                │
├─────────────────────────────────────────────────────────────┤
│ • Capture connect parameters                                │
│ • Store client metadata (user agent, IP, etc.)             │
│ • Assign connection ID                                      │
│ • Add to clients Set                                        │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ server-methods.ts: handleGatewayRequest()                   │
├─────────────────────────────────────────────────────────────┤
│ • Extract method name from request frame                    │
│ • Check role-based authorization:                           │
│   - Admin methods require operator.admin scope             │
│   - Read methods require operator.read scope               │
│   - Write methods require operator.write scope             │
│   - Pairing methods require operator.pairing scope         │
│   - Approval methods require operator.approvals scope      │
│ • Look up handler:                                          │
│   - Check extraHandlers (plugins) first                    │
│   - Fallback to coreGatewayHandlers                        │
│ • Validate method is in allowed gateway methods list       │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Method Handler Execution                                     │
├─────────────────────────────────────────────────────────────┤
│ Handler receives:                                            │
│ • Request payload (method-specific parameters)              │
│ • Request context with access to:                           │
│   - Config (loadConfig)                                     │
│   - Cron service                                            │
│   - Exec approval manager                                   │
│   - Node registry                                           │
│   - Channel manager                                         │
│   - Broadcast function                                      │
│   - Chat run state                                          │
│   - Session utilities                                       │
│   - Plugin registry                                         │
│   - Dependencies (CLI deps)                                 │
│   - Workspace directory                                     │
│   - Skills cache                                            │
│                                                              │
│ Handler executes business logic:                            │
│ • Access shared state                                       │
│ • Perform operations                                        │
│ • Update state                                              │
│ • Trigger broadcasts if needed                             │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Response Generation                                          │
├─────────────────────────────────────────────────────────────┤
│ Handler calls respond():                                     │
│ • respond(ok=true, payload, error=null, meta)              │
│ • respond(ok=false, payload=null, error, meta)             │
│                                                              │
│ Response frame structure:                                    │
│ • id: correlation ID (matches request)                     │
│ • ok: success/failure boolean                              │
│ • payload: method-specific response data                   │
│ • error: error details if ok=false                         │
│ • meta: optional metadata                                  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Send Response to Client                                      │
├─────────────────────────────────────────────────────────────┤
│ • Serialize response as JSON                                │
│ • Send over WebSocket connection                            │
│ • Log response (if logging enabled)                         │
└─────────────────────────────────────────────────────────────┘
```

### HTTP Request Flow

```
┌─────────────────────────────────────────────────────────────┐
│ Client sends HTTP request                                   │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ server-http.ts: HTTP Route Matching                         │
├─────────────────────────────────────────────────────────────┤
│ Routes:                                                      │
│ • POST /v1/chat/completions → openai-http.ts               │
│ • POST /v1/responses → openresponses-http.ts               │
│ • POST /tools/invoke → tools-invoke-http.ts                │
│ • POST /hooks/* → hooks.ts                                 │
│ • GET /control-ui/* → control-ui.ts (static assets)        │
│ • GET /avatar/* → server-methods/agents.ts (avatars)       │
│ • GET /canvas-host/* → Canvas host handler                 │
│ • Upgrade to WebSocket → ws-connection.ts                  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Endpoint Handler Processing                                  │
├─────────────────────────────────────────────────────────────┤
│ • Parse request body (JSON)                                 │
│ • Validate request schema                                   │
│ • Authenticate (token in Authorization header)             │
│ • Execute endpoint logic                                    │
│ • Generate response                                         │
│ • Send HTTP response                                        │
└─────────────────────────────────────────────────────────────┘
```

## Key Subsystems

### Authentication & Security

**Files:** `auth.ts`, `auth-rate-limit.ts`, `device-auth.ts`

**Auth Modes:**
1. **None** - No authentication (loopback only by default)
2. **Token** - Bearer token or `?token=` query parameter
3. **Password** - Hashed password with salt (bcrypt)
4. **Device Token** - Public key signature verification
5. **Tailscale** - Automatic Tailnet user identity
6. **Trusted Proxy** - X-Forwarded-For from trusted IP

**Security Features:**
- Rate limiting for brute force protection
- TLS/HTTPS support with certificate management
- Client certificate verification for device pairing
- Role-based authorization (admin/read/write/pairing/approvals)
- Session-based access control

### WebSocket Communication

**Files:** `server-ws-runtime.ts`, `server/ws-connection.ts`, `server/ws-connection/message-handler.ts`

**Features:**
- Incremental message buffering with size limits (default 100MB max)
- Automatic frame validation against protocol schema
- Correlation IDs for request-response matching
- Connection keepalive via ping/pong
- Graceful connection close with reason codes
- Broadcast to all connected clients
- Targeted broadcast to specific connection IDs

**Message Types:**
- Request frames (client → server)
- Response frames (server → client)
- Event frames (server → client, broadcast)

### Chat & Agent Execution

**Files:** `server-chat.ts`, `server-methods/chat.ts`

**Features:**
- Track active chat runs per session
- Stream chat deltas to connected clients
- Handle streaming abort via AbortController
- Deduplicate identical chat runs within 3-5 seconds
- Buffer chat deltas for efficient streaming
- Support for multiple simultaneous chat sessions
- Chat history retrieval

**Agent Event Handling:**
- Listen to agent lifecycle events
- Broadcast chat events to connected clients
- Support for tool events and streaming
- Heartbeat visibility control

### Node Communication

**Files:** `node-registry.ts`, `node-invoke-*.ts`, `server-node-subscriptions.ts`, `server-mobile-nodes.ts`

**Node Registry:**
- Track connected agent nodes
- Store node capabilities and metadata
- Manage node presence with heartbeat
- Support for mobile nodes (iOS, Android)

**RPC-Style Communication:**
- Invoke remote node methods
- Approval workflow for sensitive operations
- Result streaming back to gateway
- Node event subscriptions (session-based)
- Remote skill discovery and caching

### Plugin System

**Files:** `server-plugins.ts`, `server-methods/`, plugin hooks

**Plugin Architecture:**
- Dynamic method registration via plugins
- Hook runner for lifecycle events:
  - `gateway_start` - On gateway startup
  - `gateway_stop` - On gateway shutdown
  - Custom hooks defined by plugins
- Plugin services for async background work
- Channel plugin integration
- HTTP route registration by plugins
- Tool registration by plugins

**Plugin Discovery:**
- Scan plugin directories
- Load plugin manifests
- Validate plugin compatibility
- Auto-enable based on environment

### Configuration Management

**Files:** `config-reload.ts`, `server-reload-handlers.ts`, `server-runtime-config.ts`

**Hot Reload:**
- File watcher on `config.json`
- Selective reload of configuration sections:
  - Cron jobs (add/remove/modify)
  - Heartbeat configuration
  - Hooks configuration
  - Channel configuration
- Graceful restart when deep changes required
- Defer reload if operations pending

**Reload Handlers:**
- Validation before applying changes
- State transition management
- Broadcast config change events
- Minimal disruption to active connections

### Health & Maintenance

**Files:** `server-maintenance.ts`, `server/health-state.ts`

**Health Monitoring:**
- Periodic health check broadcasts (~every 5 seconds)
- Health cache for fast responses
- Presence snapshot updates
- System-wide presence tracking
- Gateway status aggregation

**Maintenance Tasks:**
- Deduplication cleanup (stale chat runs)
- Node presence timeout detection
- Connection pool cleanup
- Memory optimization

**Timers:**
- Health broadcast interval
- Tick broadcast interval
- Presence update interval
- Cleanup interval

### Cron Service

**Files:** `server-cron.ts`

**Features:**
- Scheduled job execution
- Cron expression parsing
- Job history tracking
- Status monitoring
- Dynamic job management (add/remove/modify)
- Integration with hot-reload

### Channel Management

**Files:** `server-channels.ts`, channel plugins in `src/channels/plugins/`

**Supported Channels:**
- Telegram
- Discord
- Slack
- Signal
- iMessage
- WhatsApp (web)
- MS Teams (via extension)
- Matrix (via extension)
- And more via plugins

**Channel Features:**
- Unified channel abstraction
- Message routing to appropriate channel
- Status monitoring
- Connection management
- Auto-reply integration

### Discovery Services

**Files:** `server-discovery-runtime.ts`, `server-discovery.ts`, `server-tailscale.ts`

**Bonjour/mDNS:**
- Advertise gateway on local network
- Service type: `_openclaw._tcp`
- Publish gateway port and host
- Auto-discovery by clients

**Tailscale:**
- Expose gateway to Tailscale network
- Support for funnel (public) and serve (private) modes
- Automatic hostname resolution
- Identity-based authentication

### Execution Approvals

**Files:** `exec-approval-manager.ts`, `server-methods/exec-approval.ts`, `exec-approval-forwarder.ts`

**Approval Workflow:**
1. Agent requests approval for sensitive operation
2. Approval request sent to clients
3. User approves/rejects via client
4. Decision forwarded back to agent
5. Agent proceeds or aborts based on decision

**Features:**
- Timeout for pending approvals
- Approval history tracking
- Multiple approval strategies
- Remote node approval forwarding

### Session Management

**Files:** `session-utils.ts`, `session-utils.fs.ts`, `sessions-resolve.ts`, `sessions-patch.ts`

**Session Features:**
- Session storage on filesystem
- Session key resolution
- Session patching/updating
- Session listing and preview
- Session transcript management
- Session-based subscriptions

## WebSocket Protocol

### Frame Structure

**Request Frame:**
```json
{
  "id": "unique-correlation-id",
  "method": "method.name",
  "params": { /* method-specific parameters */ },
  "meta": { /* optional metadata */ }
}
```

**Response Frame:**
```json
{
  "id": "correlation-id-from-request",
  "ok": true,
  "payload": { /* method-specific response */ },
  "error": null,
  "meta": { /* optional metadata */ }
}
```

**Event Frame:**
```json
{
  "event": "event.name",
  "payload": { /* event-specific data */ }
}
```

### Gateway Methods

The gateway exposes 60+ methods organized into categories:

**Connection & Health:**
- `connect` - Initial connection handshake
- `health` - Gateway health status
- `system-presence` - System presence information
- `last-heartbeat` - Last heartbeat timestamp

**Chat & Messaging:**
- `send` - Send a message to an agent
- `agent` - Start agent conversation
- `agent.wait` - Wait for agent response
- `chat.send` - Send chat message
- `chat.abort` - Abort ongoing chat
- `chat.history` - Retrieve chat history
- `wake` - Wake voice-activated agent

**Configuration:**
- `config.get` - Get configuration
- `config.patch` - Update configuration
- `config.reload` - Reload configuration

**Sessions:**
- `sessions.list` - List sessions
- `sessions.preview` - Preview session
- `sessions.send` - Send to session

**Nodes:**
- `node.list` - List connected nodes
- `node.describe` - Get node details
- `node.invoke` - Invoke method on remote node
- `node.pair.*` - Node pairing operations
- `node.rename` - Rename a node

**Devices:**
- `device.pair.*` - Device pairing operations
- `device.token.*` - Device token management

**Channels:**
- `channels.status` - Channel status

**Models:**
- `models.list` - List available models

**Agents:**
- `agents.list` - List agents
- `agent.identity.get` - Get agent identity

**Skills:**
- `skills.status` - Skills status
- `skills.bins` - Remote skills binaries

**Cron:**
- `cron.list` - List cron jobs
- `cron.status` - Cron job status
- `cron.runs` - Cron run history

**Usage & Monitoring:**
- `usage.status` - Usage statistics
- `usage.cost` - Cost information
- `logs.tail` - Tail logs

**TTS (Text-to-Speech):**
- `tts.*` - TTS operations

**Voice Wake:**
- `voicewake.*` - Voice wake configuration

**Talk Mode:**
- `talk.*` - Talk mode configuration

**Browser Control:**
- `browser.request` - Browser control requests

**System:**
- `update.*` - Update operations
- `wizard.*` - Onboarding wizard

**Exec Approvals:**
- `exec.approval.*` - Execution approval operations
- `exec.approvals.*` - Approval management

## HTTP Endpoints

### OpenAI-Compatible Chat API

**Endpoint:** `POST /v1/chat/completions`

**File:** `openai-http.ts`

**Purpose:** OpenAI-compatible chat completions endpoint for third-party integrations

**Features:**
- Compatible with OpenAI API format
- Streaming support via SSE
- Model selection
- Temperature, max_tokens, etc.

### OpenResponses API

**Endpoint:** `POST /v1/responses`

**File:** `openresponses-http.ts`

**Purpose:** Custom responses API for simplified agent interaction

**Features:**
- Simpler API than OpenAI format
- Session-based conversations
- Streaming support
- Built for OpenClaw-specific use cases

### Webhooks

**Endpoint:** `POST /hooks/*`

**File:** `hooks.ts`, `server/hooks.ts`

**Purpose:** Webhook integrations for external services

**Features:**
- Dynamic hook registration
- Hook authentication
- Request logging
- Integration with hook handlers

### Tool Invocation

**Endpoint:** `POST /tools/invoke`

**File:** `tools-invoke-http.ts`

**Purpose:** HTTP-based tool invocation

**Features:**
- Invoke tools via HTTP
- Authentication required
- Result streaming

### Control UI

**Endpoint:** `GET /control-ui/*`

**File:** `control-ui.ts`

**Purpose:** Serve web-based control UI

**Features:**
- Static asset serving
- SPA routing support
- Development mode with live reload

### Avatar Service

**Endpoint:** `GET /avatar/*`

**File:** `server-methods/agents.ts`

**Purpose:** Serve agent avatars

### Canvas Host

**Endpoint:** `GET /canvas-host/*`

**File:** Canvas host handler

**Purpose:** Serve canvas host UI for agent interactions

## State Management

### Shared State

The gateway maintains several pieces of shared state accessible to all handlers:

**Client Connections:**
- `clients: Set<GatewayWsClient>` - All active WebSocket connections

**Chat State:**
- `chatRunState` - Active chat runs per session
- `chatRunBuffers` - Buffered chat deltas for streaming
- `chatDeltaSentAt` - Last delta send timestamp per run
- `chatAbortControllers` - Abort controllers for chat runs
- `dedupe` - Deduplication map for identical chat runs

**Node State:**
- `nodeRegistry` - Connected agent nodes
- `nodePresenceTimers` - Presence heartbeat timers
- `nodeSubscriptions` - Event subscriptions per session

**Services:**
- `cronService` - Cron job manager
- `channelManager` - Channel connection manager
- `execApprovalManager` - Execution approval manager
- `heartbeatRunner` - Health heartbeat runner

**Configuration:**
- `cfg` - Current gateway configuration (hot-reloadable)
- `resolvedAuth` - Resolved authentication config
- `hooksConfig` - Hooks configuration

**Broadcast:**
- `broadcast()` - Broadcast event to all clients
- `broadcastToConnIds()` - Broadcast to specific clients

### State Synchronization

**Presence Version:**
- Incremented on presence changes
- Included in broadcast messages
- Used by clients to detect stale state

**Health Version:**
- Incremented on health changes
- Included in health broadcasts
- Used for cache validation

## Plugin System

### Plugin Structure

Plugins can provide:

1. **Gateway Methods** - New WebSocket methods
2. **HTTP Routes** - New HTTP endpoints
3. **Hooks** - Lifecycle event handlers
4. **Tools** - New agent tools
5. **Channels** - New messaging channels
6. **Providers** - New service providers
7. **Services** - Background services
8. **CLI Commands** - New CLI commands

### Plugin Registration

**File:** `server-plugins.ts`

**Process:**
1. Scan plugin directories
2. Load plugin manifests
3. Validate plugin structure
4. Register plugin components
5. Merge with core handlers

### Plugin Hooks

**Lifecycle Hooks:**
- `gateway_start` - Called after gateway starts
- `gateway_stop` - Called before gateway stops

**Custom Hooks:**
- Plugins can define and trigger custom hooks
- Other plugins can listen to these hooks

### Plugin Services

**File:** `src/plugins/services.ts`

Background services that:
- Run independently of requests
- Have access to gateway state
- Can be started/stopped
- Support graceful shutdown

## Security & Authentication

### Authentication Modes

**1. None (Loopback Only)**
- No authentication required
- Only allows connections from 127.0.0.1
- Suitable for local development

**2. Token**
- Bearer token in Authorization header
- Query parameter: `?token=xxx`
- Simple shared secret

**3. Password**
- Hashed password with bcrypt
- Stored in config file
- Protects against plaintext exposure

**4. Device Token**
- Public key cryptography
- Device registers public key
- Requests signed with private key
- Gateway verifies signature

**5. Tailscale**
- Automatic authentication via Tailnet
- User identity from Tailscale
- No additional credentials needed

**6. Trusted Proxy**
- For reverse proxy scenarios
- Trusts X-Forwarded-For from trusted IPs

### Authorization Scopes

**operator.admin** - Full control
- Config changes
- Approval management
- System operations

**operator.read** - Read-only access
- View health
- List sessions
- View configuration

**operator.write** - Standard operations
- Send messages
- Start agents
- Invoke nodes

**operator.pairing** - Device pairing
- Pair new devices
- Approve pairings
- Manage device tokens

**operator.approvals** - Execution approvals
- Request approvals
- Receive approval requests
- Resolve approval decisions

### Rate Limiting

**File:** `auth-rate-limit.ts`

**Features:**
- Configurable window (default 15 minutes)
- Configurable max attempts (default 5)
- Per-IP address tracking
- Blocks after threshold
- Auto-reset after window

### TLS/HTTPS

**File:** `server/tls.ts`

**Features:**
- Support for TLS certificates
- Key and cert file loading
- Optional CA cert chain
- HTTPS server creation
- Client certificate verification

## Summary

The OpenClaw Gateway is a sophisticated, multi-protocol server that coordinates all aspects of the OpenClaw system. Its initialization sequence follows a precise order to ensure all components are properly configured before accepting connections. The request handling flow provides robust authentication, authorization, and routing to over 60 methods. Key subsystems like chat management, node communication, plugin integration, and configuration hot-reloading work together to provide a powerful and flexible agent orchestration platform.

The architecture emphasizes:
- **Modularity** - Clear separation of concerns
- **Extensibility** - Plugin system for custom functionality
- **Reliability** - Health monitoring and graceful error handling
- **Security** - Multiple authentication mechanisms and role-based access
- **Performance** - Efficient broadcast, streaming, and deduplication
- **Flexibility** - Hot-reload configuration without downtime
