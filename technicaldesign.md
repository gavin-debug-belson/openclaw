# OpenClaw Technical Design Document

## Table of Contents

1. [System Overview](#system-overview)
2. [Architecture Layers](#architecture-layers)
3. [Core Components](#core-components)
4. [Technology Stack](#technology-stack)
5. [Directory Structure](#directory-structure)
6. [Data Flow](#data-flow)
7. [Channel Integrations](#channel-integrations)
8. [Agent Runtime](#agent-runtime)
9. [Plugin System](#plugin-system)
10. [Build & Deployment](#build--deployment)
11. [Configuration & Storage](#configuration--storage)
12. [Security Model](#security-model)

---

## System Overview

### What is OpenClaw?

OpenClaw is a **personal AI assistant platform** that runs on your own devices (like a personal server or computer). Think of it as your own private AI assistant that:

- Lives on your hardware (not in someone else's cloud)
- Works across multiple messaging apps (WhatsApp, Telegram, Slack, etc.)
- Can see what you see and help with tasks
- Stays always available and learns from your conversations
- Respects your privacy because it's yours

### Key Characteristics

**Single-User Design**: Unlike most AI services that serve millions of users, OpenClaw is designed for just one person - you. It's like having a personal butler instead of calling a customer service center.

**Self-Hosted**: You run it on your own computer or device. This means:
- Your data stays with you
- No monthly subscription fees (except for AI model APIs)
- Full control over what it does and doesn't do

**Multi-Channel**: The same AI assistant can talk to you through:
- WhatsApp on your phone
- Slack at work
- Terminal/command line on your computer
- Voice on iOS/macOS
- Web browser
- And many more...

**Always-On**: Once set up, it runs in the background waiting for your messages or commands.

---

## Architecture Layers

Think of OpenClaw like a building with multiple floors, where each floor has a specific job:

```
┌─────────────────────────────────────────────────────┐
│  LAYER 1: User Interface Layer                      │
│  (Where you interact)                               │
│  - Native Apps (iOS, Android, macOS)                │
│  - Web Browser Interface                            │
│  - Command Line (CLI)                               │
│  - Voice Interfaces                                 │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│  LAYER 2: Gateway & Routing Layer                   │
│  (Central control and message routing)              │
│  - WebSocket Server                                 │
│  - Session Management                               │
│  - Authentication                                   │
│  - Message Routing                                  │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│  LAYER 3: Channel Integration Layer                 │
│  (Connects to messaging apps)                       │
│  - WhatsApp, Telegram, Slack, Discord               │
│  - Signal, iMessage, Teams, etc.                    │
│  - Protocol Adapters                                │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│  LAYER 4: Agent Runtime Layer                       │
│  (The AI "brain")                                   │
│  - Model Selection & Management                     │
│  - Conversation Context                             │
│  - Tool Execution                                   │
│  - Memory Management                                │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│  LAYER 5: AI Model Provider Layer                   │
│  (External AI services)                             │
│  - Claude (Anthropic)                               │
│  - GPT (OpenAI)                                     │
│  - AWS Bedrock                                      │
│  - Local Models (Ollama)                            │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│  LAYER 6: Tools & Skills Layer                      │
│  (What the AI can actually do)                      │
│  - Run commands (bash)                              │
│  - Browse the web                                   │
│  - Edit files                                       │
│  - Search code                                      │
│  - And 40+ other skills                             │
└─────────────────────────────────────────────────────┘
```

### How Layers Work Together

Imagine you send a message "What's the weather?" to OpenClaw via WhatsApp:

1. **Layer 1**: Your phone's WhatsApp app sends the message
2. **Layer 2**: The Gateway receives it and figures out which conversation this belongs to
3. **Layer 3**: The WhatsApp channel adapter processes the message
4. **Layer 4**: The Agent Runtime decides which AI model to use and asks it to respond
5. **Layer 5**: Claude (or another AI) generates a response and says "I need to use the weather tool"
6. **Layer 6**: The weather skill fetches current weather data
7. **Layers go back up**: The response flows back through the layers to your WhatsApp

---

## Core Components

### 1. Gateway (The Central Hub)

**Location**: `src/gateway/`

**What It Does**: The Gateway is like a traffic controller at an airport. It:
- Receives messages from all your apps and channels
- Keeps track of who is talking and when
- Routes messages to the right place
- Manages security and authentication
- Stores conversation history

**Key Parts**:
- **Server** (`server.ts`): The main program that runs 24/7
- **Routes** (`routes.ts`): Defines what happens for different types of requests
- **Session Manager** (`sessions.ts`): Tracks all your active conversations
- **WebSocket Handler**: Enables real-time two-way communication

**How It Works**:
1. Starts a server on your computer (usually port 18789)
2. Listens for connections from apps and channels
3. When a message arrives, it figures out which conversation it belongs to
4. Passes the message to the Agent Runtime for processing
5. Sends the AI's response back to the original channel

### 2. Agent Runtime (The AI Brain)

**Location**: `src/agents/`

**What It Does**: This is where the "thinking" happens. It:
- Picks which AI model to use (Claude, GPT, etc.)
- Manages the conversation context (remembering what you talked about)
- Decides when to use tools (like running code or searching the web)
- Handles errors and retries if something fails
- Streams responses back as they're generated

**Key Parts**:
- **Pi Embedded Runner** (`pi-embedded-runner.ts`): The main orchestrator
- **Model Selection** (`model-selection.ts`): Chooses the best AI model for the task
- **Auth Profiles** (`auth-profile-manager.ts`): Manages API keys for different AI services
- **Tool Registry** (`tools/`): Registry of all available tools the AI can use
- **Context Manager**: Keeps track of conversation history

**How It Works**:
1. Receives a message from the Gateway
2. Loads the conversation history (context)
3. Selects an AI model based on the task and available API keys
4. Sends the message to the AI model with available tools
5. If the AI wants to use a tool (like "search the web"), it executes it
6. Streams the AI's response back to the Gateway

### 3. Channel Integrations (Message App Connectors)

**Location**: `src/[channel]/` and `extensions/`

**What It Does**: Each channel integration is like a translator that speaks the language of a specific messaging app. For example:
- **WhatsApp** (`src/whatsapp/`): Uses the Baileys library to connect to WhatsApp Web
- **Telegram** (`src/telegram/`): Uses the Grammy framework for Telegram Bot API
- **Slack** (`extensions/slack/`): Uses Slack's Bolt SDK for Slack workspaces
- **Discord** (`src/discord/`): Connects to Discord servers via their API

**Key Parts**:
- **Connection Logic**: How to connect to the service (webhooks, polling, WebSocket)
- **Message Parser**: Converts the service's message format to OpenClaw's internal format
- **Message Sender**: Converts OpenClaw's responses back to the service's format
- **Authentication**: Handles logging in and staying connected

**How It Works** (WhatsApp example):
1. Connects to WhatsApp Web (like how WhatsApp Web works in your browser)
2. Listens for incoming messages
3. When a message arrives, converts it to OpenClaw's standard format
4. Sends it to the Gateway
5. When a response comes back, converts it to WhatsApp's format and sends it

### 4. Command-Line Interface (CLI)

**Location**: `src/cli/` and `src/commands/`

**What It Does**: The CLI is how you control OpenClaw from your computer's terminal/command prompt. It's like the control panel for the system.

**Main Commands**:

```bash
# Set up OpenClaw for the first time
openclaw onboard

# Start the Gateway server
openclaw gateway run

# Talk to the AI from the terminal
openclaw agent --message "Hello"

# Send a message through a channel
openclaw telegram send "What's the weather?"

# Configure settings
openclaw config set gateway.port 8080

# Manage the background service
openclaw daemon install
openclaw daemon start
openclaw daemon stop

# Interactive chat in terminal
openclaw tui
```

**How It Works**:
1. You type a command in your terminal
2. The CLI program (`openclaw.mjs`) receives it
3. It parses the command and options
4. Calls the appropriate function in `src/commands/`
5. That function does the work (starts a server, sends a message, etc.)
6. Results are displayed back in your terminal

### 5. Memory & Storage

**Location**: `src/memory/` and `src/config/`

**What It Does**: This handles all the data that needs to be saved:
- **Conversation History**: What you and the AI talked about
- **Configuration**: Your settings and preferences
- **Session Data**: Active conversations and their state
- **Embeddings**: Mathematical representations of text for semantic search

**Storage Locations**:
- Configuration: `~/.openclaw/config.yaml`
- Credentials: `~/.openclaw/credentials/`
- Sessions: `~/.openclaw/sessions/`
- Logs: `~/.openclaw/logs/`

**How It Works**:
1. When you have a conversation, it's saved to a session file
2. Configuration is loaded when OpenClaw starts
3. Memory can be searched semantically (finding related conversations)
4. Everything is stored locally on your device

### 6. Plugin System

**Location**: `src/plugins/` and `extensions/`

**What It Does**: Allows you to extend OpenClaw with custom functionality without modifying the core code. It's like installing apps on your phone.

**Types of Plugins**:
- **Channel Plugins**: Add support for new messaging platforms
- **Tool Plugins**: Add new capabilities for the AI
- **Provider Plugins**: Add support for new AI models

**How It Works**:
1. Plugins are installed in the `extensions/` directory
2. Each plugin exports a manifest describing what it does
3. OpenClaw loads plugins at startup
4. Plugins can register new channels, tools, or providers
5. The rest of the system can use these new capabilities

---

## Technology Stack

### Core Runtime

**Node.js**: Think of this as the engine that runs the program. Node.js lets you run JavaScript (a programming language) outside of a web browser. OpenClaw requires version 22 or newer.

**TypeScript**: This is a stricter version of JavaScript that helps catch errors before the program runs. All of OpenClaw's code is written in TypeScript, then converted ("compiled") to regular JavaScript.

**ESM (ES Modules)**: This is a modern way of organizing code into separate files that can import functionality from each other.

### Key Libraries

**Express**: A framework for creating web servers. This is what the Gateway uses to handle incoming connections and requests.

**WebSocket (ws)**: A library for real-time, two-way communication between programs. This is how the Gateway talks to apps and channels.

**Commander**: A library that makes it easy to create command-line programs with options and subcommands.

**AI/LLM Libraries**:
- Anthropic SDK: For talking to Claude
- OpenAI SDK: For talking to GPT models
- AWS SDK: For AWS Bedrock models
- Custom integrations for other providers

**Messaging Libraries**:
- `@whiskeysockets/baileys`: WhatsApp Web protocol
- `grammy`: Telegram Bot API
- `@slack/bolt`: Slack app framework
- `discord-api-types`: Discord API

**Database**: 
- SQLite with vector extensions: A lightweight database stored in files
- Used for storing embeddings and searching conversations

**Media Processing**:
- `sharp`: Image processing (resize, convert formats)
- `pdfjs-dist`: Reading PDF files
- Browser automation for screenshots

### Development Tools

**pnpm**: A package manager (like an app store for code libraries). It downloads and manages all the dependencies OpenClaw needs.

**tsdown**: Converts TypeScript to JavaScript and bundles everything together.

**Vitest**: A testing framework to make sure everything works correctly.

**Oxlint & Oxfmt**: Tools that check code style and format it consistently.

---

## Directory Structure

```
openclaw/
│
├── src/                          # Main source code (TypeScript)
│   ├── gateway/                  # Central server & routing
│   ├── agents/                   # AI orchestration
│   ├── cli/                      # Command-line interface
│   ├── commands/                 # High-level operations
│   ├── channels/                 # Channel abstraction layer
│   ├── telegram/                 # Telegram integration
│   ├── discord/                  # Discord integration
│   ├── slack/                    # Slack integration (core parts)
│   ├── signal/                   # Signal integration
│   ├── imessage/                 # iMessage integration
│   ├── whatsapp/                 # WhatsApp integration
│   ├── web/                      # Web API server
│   ├── tui/                      # Terminal user interface
│   ├── config/                   # Configuration management
│   ├── memory/                   # Session storage & embeddings
│   ├── plugins/                  # Plugin SDK
│   ├── infra/                    # Infrastructure utilities
│   ├── media/                    # Media processing
│   ├── security/                 # Authentication & security
│   ├── hooks/                    # Lifecycle hooks
│   ├── providers/                # LLM provider integrations
│   ├── terminal/                 # Terminal utilities
│   ├── browser/                  # Browser automation
│   └── ...                       # Many more modules
│
├── apps/                         # Native applications
│   ├── ios/                      # iPhone/iPad app (Swift)
│   ├── android/                  # Android app (Kotlin)
│   ├── macos/                    # macOS menubar app (Swift/SwiftUI)
│   └── shared/                   # Shared code across platforms
│
├── extensions/                   # Plugin packages
│   ├── slack/                    # Slack plugin
│   ├── msteams/                  # Microsoft Teams
│   ├── googlechat/               # Google Chat
│   ├── matrix/                   # Matrix protocol
│   ├── zalo/                     # Zalo messenger
│   ├── voice-call/               # Voice calling
│   └── ...                       # 30+ more plugins
│
├── skills/                       # AI tool definitions
│   ├── calendar/                 # Calendar integration
│   ├── coding/                   # Code editing tools
│   ├── music/                    # Music control
│   ├── weather/                  # Weather information
│   └── ...                       # 40+ more skills
│
├── ui/                           # Web frontend
│   ├── src/                      # React/Lit components
│   ├── public/                   # Static assets
│   └── vite.config.ts            # Build configuration
│
├── packages/                     # Internal packages
│   ├── clawdbot/                 # Bot variant
│   └── moltbot/                  # Another bot variant
│
├── docs/                         # Documentation
│   ├── channels/                 # Channel guides
│   ├── platforms/                # Platform-specific docs
│   ├── gateway/                  # Gateway documentation
│   └── ...                       # More guides
│
├── scripts/                      # Build & utility scripts
│   ├── package-mac-app.sh        # Package macOS app
│   ├── committer                 # Git commit helper
│   └── ...                       # More scripts
│
├── test/                         # Test utilities
│   ├── helpers/                  # Test helper functions
│   └── fixtures/                 # Test data
│
├── dist/                         # Compiled output (generated)
│   ├── *.js                      # JavaScript files
│   ├── *.d.ts                    # TypeScript definitions
│   └── plugin-sdk/               # Plugin SDK exports
│
├── openclaw.mjs                  # Main CLI entry point
├── package.json                  # Project metadata & dependencies
├── pnpm-workspace.yaml           # Monorepo configuration
├── tsconfig.json                 # TypeScript configuration
├── vitest.config.ts              # Test configuration
└── README.md                     # Project documentation
```

### Key Directories Explained

**`src/`**: This is where all the main code lives. Each subdirectory handles a specific part of the system:
- **gateway**: The central server
- **agents**: AI logic
- **channels**: Message routing
- **[channel name]**: Integration with specific messaging apps

**`apps/`**: Native applications for mobile and desktop:
- Built using each platform's native tools (Swift for iOS/macOS, Kotlin for Android)
- Connect to the Gateway via WebSocket
- Provide mobile-specific features (voice, camera, notifications)

**`extensions/`**: Separate packages that extend OpenClaw:
- Each is its own mini-project with its own `package.json`
- Can be installed or removed independently
- Follow the plugin API defined in `src/plugins/`

**`skills/`**: Definitions of what the AI can do:
- Each skill is a JSON/YAML file describing a tool
- Includes the tool's name, description, parameters, and how to execute it
- Loaded dynamically at runtime

**`ui/`**: The web-based user interface:
- Built with modern web technologies (Vite, React/Lit)
- Provides a dashboard for controlling OpenClaw
- Runs in your web browser

**`dist/`**: The compiled output ready to run:
- Generated by running `pnpm build`
- JavaScript files that Node.js can execute
- Not checked into version control (created during build)

---

## Data Flow

### Message Flow (User → AI → User)

Let's walk through what happens when you send a message:

```
┌─────────────────────────────────────────────────────────────┐
│ STEP 1: User Sends Message                                  │
├─────────────────────────────────────────────────────────────┤
│ You: "What's the weather in San Francisco?"                 │
│ Channel: WhatsApp                                            │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 2: Channel Receives Message                            │
├─────────────────────────────────────────────────────────────┤
│ WhatsApp integration detects new message                    │
│ Converts WhatsApp format to internal format:                │
│ {                                                            │
│   text: "What's the weather in San Francisco?",             │
│   channelId: "whatsapp",                                     │
│   conversationId: "123abc",                                  │
│   timestamp: 1234567890                                      │
│ }                                                            │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 3: Gateway Routes Message                              │
├─────────────────────────────────────────────────────────────┤
│ Gateway receives the formatted message                      │
│ Looks up conversation in session store                      │
│ Loads conversation history                                  │
│ Determines this needs the Agent Runtime                     │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 4: Agent Processes Message                             │
├─────────────────────────────────────────────────────────────┤
│ Agent Runtime receives message + history                    │
│ Selects AI model (e.g., Claude Sonnet)                      │
│ Builds prompt with:                                         │
│   - System instructions                                     │
│   - Conversation history                                    │
│   - Available tools (including weather)                     │
│   - User's message                                          │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 5: AI Model Responds                                   │
├─────────────────────────────────────────────────────────────┤
│ Claude receives the prompt                                  │
│ Decides to use the weather tool                             │
│ Returns: {                                                   │
│   tool_use: {                                                │
│     name: "weather",                                         │
│     input: { location: "San Francisco" }                    │
│   }                                                          │
│ }                                                            │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 6: Tool Execution                                      │
├─────────────────────────────────────────────────────────────┤
│ Agent finds the weather tool                                │
│ Executes: fetch weather API for San Francisco              │
│ Returns: {                                                   │
│   temperature: 65,                                           │
│   condition: "Partly Cloudy",                               │
│   humidity: 70%                                              │
│ }                                                            │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 7: AI Generates Final Response                         │
├─────────────────────────────────────────────────────────────┤
│ Agent sends tool result back to Claude                      │
│ Claude generates natural language response:                 │
│ "The weather in San Francisco is currently 65°F             │
│  and partly cloudy, with 70% humidity."                     │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 8: Response Sent Back                                  │
├─────────────────────────────────────────────────────────────┤
│ Agent streams response to Gateway                           │
│ Gateway routes to WhatsApp integration                      │
│ WhatsApp integration sends message to WhatsApp              │
│ You receive the response on your phone                      │
└─────────────────────────────────────────────────────────────┘
```

### Streaming Responses

OpenClaw supports streaming, which means you see the AI's response as it's being generated (like ChatGPT's typing effect):

1. **Chunk 1**: "The weather in"
2. **Chunk 2**: " San Francisco is"
3. **Chunk 3**: " currently 65°F"
4. **Chunk 4**: " and partly cloudy..."

Each chunk flows through the same path (Gateway → Channel → You) immediately as it's generated.

### Error Handling

If something goes wrong at any step:

1. **Tool Fails**: Agent tells AI the tool failed; AI tries another approach or reports error
2. **AI Model Fails**: Agent switches to backup model (failover)
3. **Channel Fails**: Gateway queues message for retry
4. **Gateway Fails**: Apps/channels reconnect automatically

---

## Channel Integrations

### How Channels Work

Each messaging platform has its own way of sending and receiving messages. OpenClaw provides integrations for each platform that translate between the platform's format and OpenClaw's internal format.

### Channel Types

#### 1. Webhook-Based Channels

**Examples**: Telegram, Slack, Discord (some modes)

**How They Work**:
1. You register a webhook URL with the service (e.g., `https://your-server.com/telegram`)
2. When someone sends a message, the service sends an HTTP request to your URL
3. Your server processes the request and sends a response

**Benefits**: 
- Real-time delivery
- Service handles connection management
- Works behind firewalls (if you expose a public URL)

#### 2. Polling-Based Channels

**Examples**: Some Telegram modes

**How They Work**:
1. Your program repeatedly asks the service "Do you have any new messages?"
2. The service responds with messages since the last check
3. You process the messages and send responses

**Benefits**:
- No need for a public URL
- Works on any network
- Simple to set up

#### 3. WebSocket-Based Channels

**Examples**: Discord, Native apps

**How They Work**:
1. Your program opens a persistent connection to the service
2. Messages flow both ways over this connection in real-time
3. Connection stays open until explicitly closed

**Benefits**:
- Real-time, bidirectional
- Lower latency than polling
- Efficient for high-volume messages

#### 4. Protocol-Based Channels

**Examples**: WhatsApp (Baileys), Signal, Matrix

**How They Work**:
1. Your program implements the same protocol the app uses
2. Connects to the service as if it were a native app
3. Sends and receives messages like any other client

**Benefits**:
- Full access to platform features
- No API key needed (sometimes)
- Works with platforms that don't offer APIs

### Channel Configuration

Each channel requires configuration stored in `~/.openclaw/config.yaml`:

```yaml
channels:
  telegram:
    enabled: true
    botToken: "YOUR_BOT_TOKEN"
    
  whatsapp:
    enabled: true
    sessionPath: "~/.openclaw/sessions/whatsapp"
    
  slack:
    enabled: true
    appToken: "xapp-..."
    botToken: "xoxb-..."
```

### Channel Features

Different channels support different features:

| Feature | WhatsApp | Telegram | Slack | Discord |
|---------|----------|----------|-------|---------|
| Text Messages | ✅ | ✅ | ✅ | ✅ |
| Images | ✅ | ✅ | ✅ | ✅ |
| Voice Messages | ✅ | ✅ | ❌ | ✅ |
| Files | ✅ | ✅ | ✅ | ✅ |
| Reactions | ✅ | ✅ | ✅ | ✅ |
| Threads | ❌ | ❌ | ✅ | ✅ |
| Voice Calls | ❌ | ❌ | ❌ | ✅ |

OpenClaw automatically adapts to what each channel supports.

---

## Agent Runtime

### What is the Agent Runtime?

The Agent Runtime is the "brain" of OpenClaw. It's responsible for:
- Deciding which AI model to use
- Managing conversation context
- Executing tools
- Handling errors and retries
- Streaming responses

### Model Selection

OpenClaw supports multiple AI models and automatically picks the best one:

**Available Models**:
- **Claude (Anthropic)**: Sonnet, Opus, Haiku
- **GPT (OpenAI)**: GPT-4, GPT-4 Turbo, GPT-3.5
- **AWS Bedrock**: Claude, Titan, others
- **Ollama**: Local models running on your computer
- **Custom**: Any API-compatible model

**Selection Strategy**:
1. Check which models you have API keys for
2. Consider the task complexity (simple question vs. coding task)
3. Check rate limits and availability
4. Pick the most appropriate model
5. If it fails, try the next best option (failover)

### Auth Profiles

**What Are Auth Profiles?**: Collections of API keys grouped by priority.

**Why Multiple Profiles?**:
- Different API keys have different rate limits
- Some keys might be free tier, others paid
- Rotate through keys to avoid hitting limits
- Fallback if one key stops working

**Example**:
```yaml
agents:
  authProfiles:
    - name: "primary"
      anthropic: "sk-ant-..."
      openai: "sk-..."
    - name: "backup"
      anthropic: "sk-ant-backup..."
```

### Context Management

**What is Context?**: The conversation history and system instructions the AI sees.

**Context Window**: AI models have a limit on how much text they can process at once (e.g., 200,000 characters).

**How OpenClaw Manages Context**:
1. **Stores Full History**: Every message is saved
2. **Loads Recent Messages**: Only includes recent messages in the AI prompt
3. **Summarizes Old Messages**: Can summarize older parts of the conversation
4. **Prunes When Needed**: Removes oldest messages if context is too long

### Tool Execution

**What Are Tools?**: Functions the AI can call to perform actions.

**Example Tools**:
- `bash`: Run terminal commands
- `view`: Read files
- `edit`: Modify files
- `web_search`: Search the web
- `weather`: Get weather information
- `calendar`: Manage calendar events

**Tool Execution Flow**:
1. AI decides it needs a tool
2. Agent validates the tool use is allowed
3. Agent executes the tool
4. Tool returns result
5. Agent sends result back to AI
6. AI uses result to continue its response

**Security**: Some tools require approval:
- Running commands (`bash`)
- Editing sensitive files
- Making network requests

### Streaming

**What is Streaming?**: Sending the response in small chunks as it's generated, rather than waiting for the entire response.

**Benefits**:
- Feels faster (you see progress immediately)
- Lower perceived latency
- Can stop generation early if needed

**How It Works**:
1. AI generates tokens (words/parts of words) one at a time
2. Agent receives each token
3. Agent immediately sends it to the Gateway
4. Gateway sends it to the channel
5. You see it on your device

---

## Plugin System

### What Are Plugins?

Plugins are add-ons that extend OpenClaw's functionality without modifying the core code. Think of them like browser extensions or phone apps.

### Types of Plugins

#### 1. Channel Plugins

**Purpose**: Add support for new messaging platforms

**Example**: Microsoft Teams plugin in `extensions/msteams/`

**What They Provide**:
- Connection logic
- Message parsing
- Message sending
- Authentication

**Structure**:
```
extensions/msteams/
├── package.json          # Plugin metadata
├── src/
│   ├── index.ts          # Plugin entry point
│   ├── client.ts         # Teams API client
│   └── parser.ts         # Message parser
└── README.md             # Documentation
```

#### 2. Tool Plugins

**Purpose**: Add new capabilities for the AI

**Example**: Calendar plugin

**What They Provide**:
- Tool definition (name, description, parameters)
- Execution logic

#### 3. Provider Plugins

**Purpose**: Add support for new AI models

**Example**: Custom LLM provider

**What They Provide**:
- API client
- Prompt formatting
- Response parsing
- Streaming support

### Plugin API

Plugins use the Plugin SDK (`src/plugins/plugin-sdk.ts`):

```typescript
// Plugin interface
interface Plugin {
  name: string;
  version: string;
  type: 'channel' | 'tool' | 'provider';
  
  // Initialization
  initialize(context: PluginContext): Promise<void>;
  
  // Registration
  register(): PluginCapabilities;
  
  // Cleanup
  destroy(): Promise<void>;
}
```

### Plugin Lifecycle

1. **Discovery**: OpenClaw scans the `extensions/` directory
2. **Loading**: Each plugin's entry point is loaded
3. **Initialization**: Plugin's `initialize()` method is called
4. **Registration**: Plugin registers its capabilities
5. **Runtime**: Plugin's functions are called as needed
6. **Destruction**: Plugin's `destroy()` method is called on shutdown

### Installing Plugins

```bash
# List available plugins
openclaw plugins list

# Install a plugin
openclaw plugins install @openclaw/msteams

# Enable/disable plugins
openclaw plugins enable msteams
openclaw plugins disable msteams

# Update plugins
openclaw plugins update msteams
```

### Creating a Plugin

1. Create a directory in `extensions/`
2. Add `package.json` with plugin metadata
3. Implement the Plugin interface
4. Export from `src/index.ts`
5. Add to `pnpm-workspace.yaml`
6. Build with `pnpm build`

---

## Build & Deployment

### Development Workflow

#### 1. Initial Setup

```bash
# Install dependencies
pnpm install

# Build the project
pnpm build

# Run in development mode
pnpm dev
```

#### 2. Development Commands

```bash
# Type checking
pnpm tsgo

# Linting and formatting
pnpm check

# Auto-fix formatting
pnpm format:fix

# Run tests
pnpm test

# Run tests with coverage
pnpm test:coverage
```

#### 3. Building

**What Happens During Build**:

```bash
pnpm build
```

1. **TypeScript Compilation**: Converts `.ts` files to `.js` files
2. **Bundling**: Combines related files together
3. **Type Generation**: Creates `.d.ts` files for TypeScript types
4. **Asset Copying**: Copies non-code files (images, configs)
5. **Plugin SDK**: Generates plugin SDK exports

**Output**: Everything goes into the `dist/` directory

### Monorepo Structure

OpenClaw uses a monorepo (multiple packages in one repository) managed by pnpm:

**Workspaces** (defined in `pnpm-workspace.yaml`):
```yaml
packages:
  - '.'           # Main package (openclaw)
  - 'ui'          # Web UI
  - 'packages/*'  # Additional packages
  - 'extensions/*' # All plugins
```

**Benefits**:
- Share code easily between packages
- Build everything together
- Manage dependencies centrally
- Version packages independently

### Deployment

#### CLI Tool

**Installation**:
```bash
npm install -g openclaw
```

**What Gets Installed**:
- `openclaw` command-line tool
- All compiled JavaScript from `dist/`
- Dependencies listed in `package.json`

#### Native Apps

**macOS**:
1. Build: `scripts/package-mac-app.sh`
2. Output: `OpenClaw.app` application bundle
3. Distribution: DMG file or via Sparkle updates

**iOS**:
1. Open `apps/ios/OpenClaw.xcodeproj` in Xcode
2. Build for iOS
3. Archive and upload to App Store or TestFlight

**Android**:
1. Open `apps/android/` in Android Studio
2. Build: `./gradlew assembleRelease`
3. Output: APK file
4. Distribution: Google Play Store or direct APK

#### Gateway Deployment

**Local Mode** (runs on your computer):
```bash
openclaw gateway run
```

**Daemon Mode** (runs in background):
```bash
openclaw daemon install
openclaw daemon start
```

**Remote Mode** (runs on a server):
1. Set up server (VPS, Raspberry Pi, etc.)
2. Install OpenClaw
3. Configure firewall for port 18789
4. Run gateway: `openclaw gateway run --bind 0.0.0.0`
5. Connect from apps using server's IP address

---

## Configuration & Storage

### Configuration File

**Location**: `~/.openclaw/config.yaml`

**Structure**:
```yaml
# Gateway settings
gateway:
  mode: local              # local | remote
  host: localhost
  port: 18789
  
# Agent settings
agents:
  defaultModel: claude-sonnet
  authProfiles:
    - name: primary
      anthropic: sk-ant-...
      openai: sk-...

# Channel settings
channels:
  telegram:
    enabled: true
    botToken: "..."
  whatsapp:
    enabled: true
    
# Security settings
security:
  approvals:
    bash: true             # Require approval for bash commands
    
# Voice settings
voice:
  enabled: true
  wakeWord: "hey openclaw"
```

### Storage Locations

**Configuration**:
- `~/.openclaw/config.yaml` - Main configuration
- `~/.openclaw/credentials/` - API keys and tokens

**Data**:
- `~/.openclaw/sessions/` - Conversation history
- `~/.openclaw/agents/` - Agent-specific data
- `~/.openclaw/memory/` - Long-term memory

**Logs**:
- `~/.openclaw/logs/` - Application logs
- `/tmp/openclaw-gateway.log` - Gateway log (sometimes)

**Platform-Specific**:
- macOS: `~/Library/Preferences/ai.openclaw.mac.plist`
- iOS: App sandbox storage
- Android: App internal storage

### Session Storage

**What is Stored**:
- Message history (user and AI messages)
- Tool calls and results
- Timestamps
- Metadata (model used, tokens, etc.)

**Format**: JSONL (JSON Lines) - one JSON object per line

**Example**:
```jsonl
{"role":"user","content":"Hello","timestamp":1234567890}
{"role":"assistant","content":"Hi there!","timestamp":1234567891}
{"role":"user","content":"What's the weather?","timestamp":1234567892}
```

**Benefits**:
- Easy to read and debug
- Can be processed line by line (memory efficient)
- Standard format

### Credentials Management

**Encryption**: Credentials are stored encrypted at rest (platform-dependent):
- macOS: Keychain
- iOS: Keychain
- Android: EncryptedSharedPreferences
- Linux: libsecret or encrypted files

**Access**: Only the OpenClaw process can access credentials

---

## Security Model

### Authentication

#### Gateway Authentication

**Purpose**: Ensure only authorized clients can connect to your Gateway.

**Methods**:
1. **API Keys**: Client provides a secret key
2. **JWT Tokens**: Time-limited tokens for web/app clients
3. **Certificate-Based**: Mutual TLS authentication

**Example Flow**:
1. Client connects to Gateway
2. Client sends authentication credentials
3. Gateway validates credentials
4. If valid, connection is established
5. If invalid, connection is rejected

#### Channel Authentication

Each channel has its own authentication:
- **Telegram**: Bot token from BotFather
- **WhatsApp**: QR code scan (like WhatsApp Web)
- **Slack**: OAuth flow with workspace
- **Discord**: Bot token from Discord Developer Portal

### Authorization

**Command Approvals**: Some actions require explicit approval:

```yaml
security:
  approvals:
    bash: true              # Approve bash commands
    fileEdit: true          # Approve file edits
    network: false          # Auto-allow network requests
```

**How It Works**:
1. AI wants to run a bash command
2. Agent checks if approval is required
3. If required, prompts user: "Allow AI to run `ls -la`? [y/n]"
4. User approves or denies
5. Action proceeds or is blocked

### Rate Limiting

**Purpose**: Prevent abuse and manage API costs

**Limits**:
- Messages per minute per channel
- AI API calls per hour
- Tool executions per conversation

**Example**:
```yaml
security:
  rateLimits:
    messagesPerMinute: 60
    aiCallsPerHour: 100
```

### Data Privacy

**Local-First**:
- All data stored on your device by default
- No telemetry or analytics sent to OpenClaw servers
- You control where and how data is stored

**API Keys**:
- Your AI API keys are stored securely
- Never shared with OpenClaw or third parties
- Only used to call AI services you configure

**Conversation Data**:
- Stored locally in `~/.openclaw/sessions/`
- Only sent to AI services you explicitly configure
- Can be deleted at any time

### Network Security

**TLS/SSL**: All external connections use encryption:
- Gateway WebSocket uses WSS (WebSocket Secure)
- API calls use HTTPS
- Channel connections use platform-specific encryption

**Firewall**: Gateway can bind to:
- `localhost` (127.0.0.1): Only accessible from your computer
- `0.0.0.0`: Accessible from network (use with caution)
- Specific IP: Bind to a specific network interface

---

## Appendix: Key Concepts for Non-Node Developers

### What is Node.js?

**Node.js** is a runtime that lets you run JavaScript code outside of a web browser. Normally, JavaScript only runs in browsers (like Chrome or Firefox), but Node.js lets you use it to build:
- Web servers
- Command-line tools
- Desktop applications
- And more

**Why Node.js?**:
- JavaScript is widely known
- Large ecosystem of libraries (npm)
- Good for I/O-heavy tasks (like chat applications)
- Asynchronous by default (handles many connections efficiently)

### What is npm/pnpm?

**npm** (Node Package Manager) is like an app store for code libraries. Instead of writing everything from scratch, you can install libraries that others have created.

**pnpm** is a faster, more efficient version of npm that OpenClaw uses.

**Example**:
```bash
# Install a library
pnpm install express

# This downloads the "express" library and saves it in node_modules/
```

### What is TypeScript?

**TypeScript** is JavaScript with type checking. It helps catch errors before running the code.

**JavaScript**:
```javascript
function add(a, b) {
  return a + b;
}
add(5, "10"); // Returns "510" (bug!)
```

**TypeScript**:
```typescript
function add(a: number, b: number): number {
  return a + b;
}
add(5, "10"); // ERROR: "10" is not a number
```

TypeScript is converted to JavaScript before running.

### What is Async/Await?

JavaScript (and Node.js) is asynchronous, meaning it can do multiple things at once without waiting.

**Synchronous** (waiting):
```typescript
const data = readFile('file.txt');  // Wait for file to be read
console.log(data);                   // Then print it
```

**Asynchronous** (not waiting):
```typescript
const data = await readFile('file.txt'); // Wait without blocking
console.log(data);                        // Then print it
```

The `await` keyword says "wait for this to finish, but let other things run meanwhile."

### What is a Monorepo?

A **monorepo** is one repository (folder) that contains multiple related projects.

**OpenClaw's Monorepo**:
```
openclaw/
├── openclaw (main CLI)
├── ui (web interface)
├── extensions/slack (Slack plugin)
├── extensions/teams (Teams plugin)
└── ... more projects
```

**Benefits**:
- Share code between projects
- One place for all code
- Easy to keep in sync

### What is a REST API?

**REST API** (Representational State Transfer) is a way for programs to communicate over HTTP (the same protocol web browsers use).

**Example**:
```
GET /api/sessions           # Get list of sessions
POST /api/messages          # Send a message
DELETE /api/sessions/123    # Delete a session
```

The Gateway provides a REST API for clients to interact with it.

### What is WebSocket?

**WebSocket** is like a phone call between programs: a two-way connection that stays open.

**HTTP** (normal web requests):
- Client asks a question
- Server answers
- Connection closes

**WebSocket**:
- Client connects
- Both can send messages anytime
- Connection stays open

OpenClaw uses WebSocket for real-time communication between the Gateway and clients.

---

## Conclusion

OpenClaw is a sophisticated, multi-layered system that brings AI assistance to your personal devices while maintaining privacy and control. Its architecture is designed to be:

- **Modular**: Each component has a clear responsibility
- **Extensible**: Plugins allow customization without core changes
- **Resilient**: Failover and error handling at multiple levels
- **Local-First**: Your data stays with you
- **Multi-Platform**: Works on iOS, Android, macOS, and Linux

The key to understanding OpenClaw is seeing how the layers work together:
1. **User Interface** (where you interact)
2. **Gateway** (central routing)
3. **Channels** (messaging app connectors)
4. **Agent Runtime** (AI orchestration)
5. **AI Models** (the intelligence)
6. **Tools** (what the AI can do)

Each layer is independent and replaceable, making the system flexible and maintainable.
