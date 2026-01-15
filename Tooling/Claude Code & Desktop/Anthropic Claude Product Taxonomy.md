# Table of Contents

- [[#Product Overview]]
- [[#Authentication & Backend Architecture]]
- [[#Billing Models]]
- [[#Model Selection]]
- [[#Feature Comparison Matrix]]
- [[#Protocol & Data Formats]]
- [[#Conversation Persistence]]
- [[#Interoperability Matrix]]
- [[#Windows-Specific Notes]]
- [[#Key Architectural Insights]]
- [[#Official Product Names and Classification]]
- [[#Architecture Grouping and Backend Infrastructure]]
- [[#Authentication Systems and Their Implications]]
- [[#Billing Models and Cost Structures]]
- [[#Rate Limiting Mechanisms]]
- [[#Feature Differentiation Between Products]]
- [[#Protocol and Data Format Specifications]]
- [[#Conversation Persistence and State Management]]
- [[#Interoperability and Cross-Product Access]]
- [[#Model Selection and Version Control]]
- [[#Summary of Product Relationships]]
- [[#Core Technology Layer: The Claude API]]
- [[#Product Classification Schema]]
- [[#Network Protocol Architecture]]
- [[#Authentication & Billing Matrix]]
- [[#Grouping by Shared Infrastructure]]
- [[#Technical Interoperability]]

---

# Complete Product Taxonomy for Claude

## Product Overview

[[#Table of Contents|Back to TOC]]

| Product                | Type               | Interface             | Primary Use Case                    |
| ---------------------- | ------------------ | --------------------- | ----------------------------------- |
| **Claude** (claude.ai) | Web application    | Browser GUI           | End-user chat, research, writing    |
| **Claude Desktop**     | Native desktop app | Desktop GUI           | Same as web + MCP local tools       |
| **Claude Code**        | CLI tool           | Terminal/command-line | Agentic coding workflows            |
| **Claude API**         | HTTP REST API      | Programmatic (JSON)   | Application integration, automation |

## Authentication & Backend Architecture

[[#Table of Contents|Back to TOC]]

| Product | Auth Methods | Backend Endpoint | Context Window |
| --- | --- | --- | --- |
| **Claude** | Account only (OAuth/password) | claude.ai infrastructure | 200K tokens |
| **Claude Desktop** | Account only (OAuth/password) | claude.ai infrastructure | 200K tokens |
| **Claude Code** | Account OR API key | claude.ai OR api.anthropic.com | 200K (account) / 1M (API key) |
| **Claude API** | API key only (`sk-ant-...`) | api.anthropic.com | 1M tokens |

**Critical Note**: Claude Code with both auth methods configured prioritizes API key over account login. Use `/status` to verify active authentication.

## Billing Models

[[#Table of Contents|Back to TOC]]

| Product                   | Cost Structure                    | Rate Limits                           | Shared Quota                    |
| ------------------------- | --------------------------------- | ------------------------------------- | ------------------------------- |
| **Claude**                | Fixed monthly ($0/$20/$25)        | Message count (~50-500 msg/5hr)       | Yes (across web/desktop/mobile) |
| **Claude Desktop**        | Fixed monthly ($0/$20/$25)        | Message count (~50-500 msg/5hr)       | Yes (across web/desktop/mobile) |
| **Claude Code** (account) | Fixed monthly (same subscription) | Message count (~10-800 prompts/5hr)   | Yes (shares web/desktop quota)  |
| **Claude Code** (API key) | Per-token ($3-15/M tokens)        | RPM/TPM/TPD (50 RPM, 40K TPM typical) | No                              |
| **Claude API**            | Per-token ($3-15/M tokens)        | RPM/TPM/TPD (50 RPM, 40K TPM typical) | No                              |

## Model Selection

[[#Table of Contents|Back to TOC]]

| Product            | Model Specification                                         | Version Control                    |
| ------------------ | ----------------------------------------------------------- | ---------------------------------- |
| **Claude**         | User aliases (Sonnet 4.5, Opus 4)                           | Automatic backend updates          |
| **Claude Desktop** | User aliases (Sonnet 4.5, Opus 4)                           | Automatic backend updates          |
| **Claude Code**    | Depends on auth: aliases (account) or version strings (API) | Automatic (account) / manual (API) |
| **Claude API**     | Explicit version strings (claude-sonnet-4-20250514)         | Developer-controlled               |

## Feature Comparison Matrix

[[#Table of Contents|Back to TOC]]

| Feature | Claude/Desktop | Claude Code | Claude API |
| --- | --- | --- | --- |
| **Artifacts** (React/HTML rendering) | ✓ | ✗ | ✗ |
| **Projects** (200K knowledge base) | ✓ | ✗ (local CLAUDE.md only) | ✗ |
| **Styles** (response templates) | ✓ | ✗ | ✗ |
| **Memory** (cross-conversation) | ✓ | ✗ | ✗ |
| **MCP Integration** (local tools) | ✓ (Desktop only) | ✗ | ✗ |
| **Prompt Caching** | ✗ | ✓ (API mode) | ✓ |
| **Batch API** (50% discount) | ✗ | ✓ (API mode) | ✓ |
| **Tool Use** (function calling) | ✗ | ✓ (API mode) | ✓ |
| **Sampling Parameters** (temp, top_p) | ✗ | ✓ (API mode) | ✓ |
| **Extended Thinking** | ✗ | ✓ (API mode) | ✓ |
| **Streaming Control** | Default on | ✓ (API mode) | ✓ |
| **File System Access** | ✗ | ✓ | Via custom code |
| **Terminal Execution** | ✗ | ✓ | Via custom code |
| **Git Integration** | ✗ | ✓ | Via custom code |

## Protocol & Data Formats

[[#Table of Contents|Back to TOC]]

| Product | Protocol | Request Format | Response Format |
| --- | --- | --- | --- |
| **Claude** | Proprietary WebSocket + REST | GUI interactions | Streamed text |
| **Claude Desktop** | Proprietary WebSocket + REST | GUI interactions | Streamed text |
| **Claude Code** (account) | Proprietary (OAuth) | CLI commands | Terminal text |
| **Claude Code** (API) | HTTPS POST | JSON (programmatic) | JSON/streamed |
| **Claude API** | HTTPS POST | JSON with full parameter control | JSON with content blocks |

## Conversation Persistence

[[#Table of Contents|Back to TOC]]

| Product | Storage Location | Sync Capability | Access Method |
| --- | --- | --- | --- |
| **Claude** | Anthropic servers | Cross-device (web/desktop/mobile) | GUI history |
| **Claude Desktop** | Anthropic servers | Cross-device (web/desktop/mobile) | GUI history |
| **Claude Code** | Local (`~/.claude/projects/`) | **None** - never syncs with claude.ai | `/resume` command |
| **Claude API** | None (stateless) | None - client manages history | Client implementation |

**Critical**: Claude Code conversations are **isolated locally** regardless of authentication method. Subscription auth only shares usage quota, not conversations.

## Interoperability Matrix

[[#Table of Contents|Back to TOC]]

| - | Claude/Desktop Conversations | Claude Code Conversations | API Conversations |
| --- | --- | --- | --- |
| **Claude/Desktop** | ✓ Synced | ✗ No access | ✗ No access |
| **Claude Code** | ✗ No access | ✓ Local only | ✗ No access |
| **Claude API** | ✗ No access | ✗ No access | Client-managed |

**Shared Elements**:

- Claude/Desktop: Full conversation sync + usage quota
- Claude Code (subscription mode) + Claude/Desktop: Usage quota **only** (no conversation sync)
- All others: Zero interoperability

## Windows-Specific Notes

[[#Table of Contents|Back to TOC]]

- Claude Code installation: npm or standalone binary
- MCP servers in Claude Desktop: Path/quoting differences vs Unix (use Windows-style paths in config)
- Environment variables: Use `set ANTHROPIC_API_KEY=...` in CMD or `$env:ANTHROPIC_API_KEY=...` in PowerShell

## Key Architectural Insights

[[#Table of Contents|Back to TOC]]

1. **Claude Code is the bridge product**: Only product with dual authentication, but maintains strict separation between modes
2. **No conversation portability**: Each product maintains isolated conversation storage
3. **Context window disparity**: API access provides 5x larger context (1M vs 200K tokens)
4. **Feature exclusivity**: Subscription features (Artifacts, Projects) unavailable in API; API features (caching, batching) unavailable in subscription
5. **Why no proxy usage**: Subscription products are API _consumers_, not _providers_—cannot expose endpoints for third-party tools like AnythingLLM

---

# Complete Product Taxonomy for Claude

## Official Product Names and Classification

[[#Table of Contents|Back to TOC]]

Anthropic offers four primary ways to access Claude, which fall into two fundamentally different architectural groups. The first group consists of **Claude** (the web interface) and **Claude Desktop** (the native application), both of which operate as subscription-based services with graphical interfaces and persistent conversation management. The second group includes **Claude API** (the developer infrastructure) and **Claude Code** (the command-line tool), though Claude Code uniquely bridges both groups through its dual authentication capability.

Here's how these products are officially designated: The web interface at claude.ai is simply called **Claude** or sometimes "Claude chat" in documentation. The desktop application is **Claude Desktop**. The command-line tool is **Claude Code**. The programmatic interface is **Claude API**, also referred to as the "Anthropic API" or technically as the "Messages API."

## Architecture Grouping and Backend Infrastructure

[[#Table of Contents|Back to TOC]]

Understanding where each product connects helps clarify how they differ. Claude and Claude Desktop both connect to the same proprietary infrastructure at claude.ai using account-based authentication through OAuth or password login. When you sign into either product, you're accessing the same backend system, which is why your conversations sync seamlessly between your browser and desktop app.

Claude API operates entirely separately at the api.anthropic.com endpoint. This is a RESTful HTTP service that accepts programmatic requests with API key authentication. There's no graphical interface here—developers write code that sends JSON payloads to this endpoint and receives model responses back.

Claude Code occupies a unique position because it can connect to either backend depending on how you authenticate. When you use Claude Code with an API key (by setting the ANTHROPIC_API_KEY environment variable), it connects directly to api.anthropic.com just like any other API client. However, when you authenticate using the `/login` command with your Claude subscription account, Claude Code connects to the claude.ai infrastructure instead—the same backend that powers the web and desktop interfaces. This dual capability makes Claude Code the only product that can access both systems.

The backend choice has significant technical implications. The api.anthropic.com endpoint currently supports a one million token context window, allowing you to work with extremely large codebases or documents. In contrast, the claude.ai infrastructure provides a 200,000 token context window. This difference exists because the API endpoint is optimized for developer workloads that often require processing large amounts of code or data, while the subscription interface is designed for interactive conversation with reasonable context limits.

## Authentication Systems and Their Implications

[[#Table of Contents|Back to TOC]]

The authentication method fundamentally determines your entire experience with Claude. Subscription products use account credentials—the email and password you create when signing up, or single sign-on through providers like Google. When you authenticate this way, the system issues JWT tokens that maintain your session and enable features like conversation persistence and cross-device synchronization.

API-based access requires API keys with the format "sk-ant-api03-" followed by a long random string. You generate these keys through the Anthropic Console at console.anthropic.com, and they function as bearer tokens—whoever possesses the key can make API requests charged to your account. This is why API keys should be treated as sensitive credentials and never committed to public repositories or shared openly.

Claude Code's dual authentication becomes particularly interesting here. If you set up Claude Code with both an API key in your environment variables and subscription authentication through the login flow, Claude Code will prioritize the API key. This can catch users by surprise because they might assume they're using their subscription (with its fixed monthly cost) when they're actually incurring per-token API charges. The `/status` command in Claude Code reveals which authentication method is currently active, helping you verify your billing situation.

## Billing Models and Cost Structures

[[#Table of Contents|Back to TOC]]

The subscription model provides predictable monthly costs regardless of usage intensity, though you face message quotas that reset on a rolling time window. The Free tier costs nothing but provides limited access to Sonnet with approximately fifty messages per five-hour period. Pro subscribers pay twenty dollars monthly and receive expanded access with roughly five hundred messages per five-hour window on more capable models. Team subscriptions at twenty-five dollars per user add collaboration features. When you hit your quota limits, the system may either prevent further messages until your quota refreshes or gracefully degrade to lower-tier models.

API billing operates on pure consumption metering. You pay only for the tokens you actually use, with separate rates for input tokens (the content you send) and output tokens (the content Claude generates). For example, Claude Sonnet 4 currently costs three dollars per million input tokens and fifteen dollars per million output tokens. This model scales perfectly with usage—if you send one request, you pay for one request. If you send a million requests, you pay proportionally. There are no monthly fees, but also no ceiling on costs if your usage spikes unexpectedly.

Claude Code's subscription mode shares the same quota system as claude.ai and Claude Desktop. This means your fifty Claude Code prompts and your interactions through the web interface all count against the same quota pool. According to Anthropic's documentation, Pro subscribers typically get somewhere between ten to forty Claude Code prompts per five-hour period, though the exact count depends on prompt complexity and length. Max 5x subscribers at one hundred dollars monthly receive approximately fifty to two hundred prompts, while Max 20x subscribers at two hundred dollars monthly get around two hundred to eight hundred prompts.

## Rate Limiting Mechanisms

[[#Table of Contents|Back to TOC]]

Subscription products enforce rate limiting through message count quotas. The system tracks how many messages you've sent in a rolling time window and prevents additional messages once you exceed your tier's limit. The limitation feels natural in an interactive chat interface—you simply see a message indicating you've reached your limit and when you can continue.

API rate limiting uses more technical metrics: requests per minute (RPM), tokens per minute (TPM), and tokens per day (TPD). A typical Build tier account might have a limit of fifty requests per minute and forty thousand tokens per minute. When you exceed these limits, the API returns an HTTP 429 status code with a "retry-after" header telling you how long to wait. This approach suits programmatic access where your code needs to handle rate limit errors gracefully and implement backoff strategies.

## Feature Differentiation Between Products

[[#Table of Contents|Back to TOC]]

The subscription interface provides several features that simply don't exist in the API realm. Artifacts allow Claude to generate interactive content like React components, HTML documents, or SVG graphics that render directly in your browser. You can see and interact with the generated code in real-time. Projects give you persistent knowledge bases where you can upload up to 200,000 tokens of reference material—style guides, API documentation, codebase samples—and Claude automatically injects this context into every conversation within that project. Styles let you choose predefined response formatting templates that adjust how Claude structures its answers. Memory enables Claude to retain information about you and your preferences across multiple conversations, creating a more personalized experience over time.

The API offers an entirely different set of capabilities optimized for programmatic integration. Prompt caching allows you to mark portions of your prompt as cacheable, dramatically reducing costs and latency for repeated content—you get a ninety percent discount on cached input tokens. The Batch API enables you to submit large numbers of requests for asynchronous processing at a fifty percent discount compared to synchronous requests. Tool use, also called function calling, lets you define JSON schemas for external functions, and Claude generates structured requests for your code to execute and return results. Extended thinking provides verbose chain-of-thought reasoning where you can allocate a specific token budget for Claude's internal analysis before generating its final response. You also get direct control over sampling parameters like temperature, top_p, and top_k, allowing fine-grained adjustment of output randomness and diversity.

Claude Desktop adds one unique capability to the subscription interface: Model Context Protocol (MCP) integration. MCP allows Claude Desktop to connect to local tools and services running on your computer—databases, file systems, development tools—enabling Claude to read and interact with local resources. This gives Claude Desktop programmatic capabilities that the browser-based interface cannot match due to security constraints. However, MCP in Claude Desktop differs fundamentally from tool use in the API. MCP servers are predefined integrations you configure, while API tool use lets you define arbitrary functions dynamically in each request.

Claude Code provides terminal-native development workflows that neither the subscription interface nor direct API access offers. It maintains awareness of your Git repository state, can execute terminal commands with your confirmation, and provides specialized slash commands for common development tasks. The `/resume` command lets you continue previous coding sessions stored locally on your machine. The `/edit` command can directly modify files in your project directory. These capabilities make Claude Code particularly powerful for software development tasks where staying in the terminal improves productivity.

## Protocol and Data Format Specifications

[[#Table of Contents|Back to TOC]]

The subscription products communicate through proprietary protocols. Claude and Claude Desktop use WebSocket connections for real-time streaming of responses, combined with HTTPS REST endpoints for operations like file uploads and authentication. The exact message format is not publicly documented and is not designed for third-party integration. Users interact through graphical interfaces—clicking buttons, typing in text boxes, uploading files through dialog boxes.

The Claude API uses standard HTTPS POST requests to the api.anthropic.com/v1/messages endpoint with JSON request and response bodies. The request structure is fully documented, giving you complete control over parameters. You specify the model version explicitly using strings like "claude-sonnet-4-20250514", construct a messages array with role and content fields, optionally provide a system prompt, and can include sampling parameters like temperature or max_tokens. The response returns a content array that might contain text blocks, tool use requests, or other content types depending on your request configuration.

Claude Code operates through a command-line interface with a conversational model. You launch it with commands like `claude "implement user authentication"` or enter an interactive mode where you type naturally and Claude responds. Behind the scenes, when using API authentication, Claude Code translates your conversational requests into proper API calls to api.anthropic.com. When using subscription authentication, it communicates with the claude.ai infrastructure using OAuth tokens, though the exact protocol mirrors the web interface's proprietary format.

## Conversation Persistence and State Management

[[#Table of Contents|Back to TOC]]

In the subscription ecosystem, every conversation you have through Claude or Claude Desktop persists on Anthropic's servers. You can access the same conversation from your phone, your laptop, your office computer—wherever you're signed in. The system maintains complete conversation history indefinitely, allowing you to search past interactions and continue old conversations. This creates a seamless multi-device experience where your context follows you everywhere.

The Claude API maintains no conversation state whatsoever. Every API request is completely independent. If you want to maintain a conversation, you must manually include the entire conversation history in each subsequent request by building up a messages array. This stateless design gives you complete control over context management and allows you to implement custom storage solutions, but it also means you're responsible for all session management.

Claude Code stores conversations locally at a specific file path on your computer: `~/.claude/projects/[project-hash]/[session-id].jsonl`. These conversations never synchronize with claude.ai, even when you authenticate using your subscription account. This means a conversation you have in Claude Code remains isolated in your terminal environment. You cannot view it on claude.ai, nor can you continue a claude.ai conversation in Claude Code. The isolation is complete—the only thing shared between Claude Code and claude.ai when using subscription authentication is your usage quota, not your conversation history. Multiple feature requests on GitHub ask for cross-platform conversation continuity, but this capability doesn't currently exist.

## Interoperability and Cross-Product Access

[[#Table of Contents|Back to TOC]]

The subscription products and API products operate as completely separate ecosystems with zero interoperability between them. Conversations you have through claude.ai or Claude Desktop never appear in API responses and cannot be accessed programmatically through API calls. Conversely, conversations conducted through the API or tools built with the API don't appear in your claude.ai conversation history.

This separation extends to authentication credentials. Your claude.ai account username and password cannot generate API keys. Your API keys cannot log you into claude.ai or Claude Desktop. They're maintained as entirely distinct credential systems with different provisioning mechanisms. If you want both subscription access and API access, you need both a subscription account and separately created API keys, managed through different dashboards.

Claude Code's dual authentication capability doesn't bridge this gap—it simply allows Claude Code to connect to either system independently. When you use subscription authentication in Claude Code, you're accessing the claude.ai ecosystem with all its limitations and features. When you use API key authentication, you're accessing the api.anthropic.com ecosystem with its different constraints and capabilities. But these remain separate—you can't somehow merge conversations or transfer context between the two modes.

## Model Selection and Version Control

[[#Table of Contents|Back to TOC]]

Subscription products present model selection through user-friendly aliases. You choose between options like "Claude Sonnet 4.5" or "Claude Opus 4" through a dropdown menu in the interface. The system automatically manages which specific model version you're using behind the scenes, and Anthropic can update to newer snapshots transparently without requiring any action from you. This abstraction simplifies the user experience but removes control over exact model versions.

The API requires explicit version strings in every request. When you want Claude Sonnet 4, you must specify the exact snapshot like "claude-sonnet-4-20250514" in your model parameter. This precision lets you lock your application to a specific model version, preventing unexpected behavior changes when Anthropic releases updated snapshots. You control when to upgrade by changing the version string in your code. The API also provides access to legacy models that may no longer be available through the subscription interface, giving you backward compatibility options for existing applications.

## Summary of Product Relationships

[[#Table of Contents|Back to TOC]]

Claude and Claude Desktop form the core subscription experience, providing managed access to Claude through polished graphical interfaces with persistent conversations and exclusive features like Artifacts and Projects. They share infrastructure, authentication, conversations, and quota—treating them as the same product accessed through different form factors makes sense.

Claude API represents pure programmatic access with developer-oriented features, stateless operation, and consumption-based billing. It's designed for building applications and integrating Claude into larger systems.

Claude Code uniquely bridges both worlds. It provides a terminal-native development experience while allowing you to choose whether you want API billing with a larger context window or subscription billing with shared quotas. However, it maintains the strict separation—your choice determines which ecosystem you're operating in for that session, and conversations never cross between the two modes.

This architecture explains why you cannot use Claude Desktop or claude.ai as a proxy for third-party tools like AnythingLLM or Cherry Studio. Those tools need direct API access with API keys, and the subscription products are fundamentally not designed to expose that kind of programmatic interface. They're endpoint consumers, not endpoint providers.

---

# Anthropic Claude Product Architecture: Taxonomy and Technical Disambiguation

## Core Technology Layer: The Claude API

[[#Table of Contents|Back to TOC]]

**Claude API** (Anthropic Messages API)

- **Definition**: RESTful HTTP API endpoint at `https://api.anthropic.com/v1/messages`
- **Protocol**: HTTPS with Bearer token authentication
- **Function**: Stateless request-response interface accepting JSON payloads containing message arrays, system prompts, and model parameters
- **Authentication Mechanism**: API keys (format: `sk-ant-api03-...`) issued through console.anthropic.com
- **Billing Model**: Token-based consumption pricing (separate from subscription products)

This is the **foundational compute layer**. All other products ultimately invoke this API endpoint or its enterprise variants.

## Product Classification Schema

[[#Table of Contents|Back to TOC]]

### Category 1: Web-Based Interfaces (Browser Applications)

**claude.ai**

- **Type**: First-party web application (SPA - Single Page Application)
- **Access Method**: Browser navigation to [https://claude.ai](https://claude.ai)
- **Authentication**: Account-based (email/password, OAuth providers)
- **Billing Model**: Subscription tiers (Free, Pro, Team)
- **Backend Connection**: Proprietary Anthropic infrastructure that abstracts the Messages API
- **State Management**: Server-side conversation persistence, cross-device synchronization
- **Key Distinction**: Does NOT require user-managed API keys; subscription grants access

### Category 2: Native Desktop Applications

**Claude Desktop**

- **Type**: First-party Electron-based desktop application
- **Supported Platforms**: Windows (.exe installer), macOS (.dmg), Linux (AppImage/deb)
- **Authentication**: Account-based (identical credential system to claude.ai)
- **Backend Connection**: Same proprietary infrastructure as claude.ai
- **Architectural Note**: Functions as a native wrapper around web technologies; NOT a local inference engine
- **Network Dependency**: Requires continuous internet connectivity; all computation occurs server-side
- **Unique Features**: Model Context Protocol (MCP) server integration for local tool access, desktop OS integration (notifications, system tray)

### Category 3: Command-Line Developer Tools

**Claude Code**

- **Type**: First-party command-line interface (CLI) tool
- **Installation Method**: Package manager distribution (npm, pip, or standalone binary)
- **Authentication**: API key-based (uses standard Claude API keys)
- **Backend Connection**: Direct invocation of `https://api.anthropic.com/v1/messages`
- **Billing Model**: Token consumption billed to API key account (not subscription)
- **Primary Use Case**: Agentic coding workflows, terminal-based interaction, CI/CD pipeline integration
- **Key Distinction**: Bypasses web/desktop UI layers; direct API client with specialized prompt scaffolding

### Category 4: Direct API Access (Programmatic Integration)

**Claude API** (as developer product)

- **Type**: Infrastructure service consumed via HTTP libraries
- **Access Pattern**: Developer writes custom code (Python `anthropic` SDK, JavaScript, cURL, etc.)
- **Authentication**: API key in `x-api-key` or `Authorization` header
- **Billing Model**: Pay-per-token consumption
- **Use Cases**: Custom application integration, third-party tool integration (AnythingLLM, Cherry Studio), production systems

## Network Protocol Architecture

[[#Table of Contents|Back to TOC]]

### What Claude Desktop Connects To

**Endpoint Domain**: `claude.ai` infrastructure (not the public Messages API)

**Connection Characteristics**:

1. **WebSocket Protocol**: Persistent bidirectional connection for real-time streaming responses
2. **HTTPS REST API**: Authentication, conversation management, file uploads
3. **Session Management**: Token-based authentication managed by Anthropic's infrastructure

**Critical Distinction**:

- Claude Desktop → `claude.ai` backend infrastructure → Internal API routing
- Claude Code / Third-party tools → `api.anthropic.com` → Direct API consumption

These are **separate network paths** with different authentication systems.

## Authentication & Billing Matrix

[[#Table of Contents|Back to TOC]]

| Product | Auth Method | Billing | API Key Required |
| --- | --- | --- | --- |
| claude.ai | Account credentials | Subscription | No |
| Claude Desktop | Account credentials | Subscription | No |
| Claude Code | API key | Token consumption | Yes |
| Claude API | API key | Token consumption | Yes |

## Grouping by Shared Infrastructure

[[#Table of Contents|Back to TOC]]

**Group A: Subscription-Based UI Products**

- claude.ai
- Claude Desktop
- Share: Unified conversation history, subscription billing, account authentication

**Group B: Developer/API Products**

- Claude Code
- Claude API (direct integration)
- Share: API key authentication, token billing, stateless operation

## Technical Interoperability

[[#Table of Contents|Back to TOC]]

**Zero Interoperability Between Groups**:

- Conversations in Claude Desktop/claude.ai are NOT accessible via API key
- API-based conversations are NOT visible in claude.ai/Claude Desktop
- No credential translation mechanism exists (cannot use subscription to generate API tokens, or vice versa)

**Reason for Separation**: Different billing systems, different conversation storage backends, different rate limiting infrastructure

This architectural separation explains why Claude Desktop cannot function as a local proxy for third-party tools like AnythingLLM—it resides in an entirely separate product domain with incompatible authentication and session management.