# Lemon AI Agent Development Guide

## Architecture Overview

Lemon AI is a **full-stack self-evolving general AI agent framework** with three deployment modes:
- **Desktop App** (Electron): `main.js` orchestrates frontend + backend + Docker setup
- **Web Application**: Koa backend (`src/app.js`) + Vue 3 frontend (`frontend/`)
- **Containerized**: Docker-based sandbox execution via `docker-compose.yml`

**Core Flow**: User request → Koa router → AgenticAgent → Planning → Code-Act execution → Runtime (Docker/Local) → Tools → LLM → Response stream

### Key Components

1. **Backend (Koa + Node.js)**
   - Entry: `bin/www` starts HTTP server on port 3000
   - App config: `src/app.js` with module-alias (`@src/`) for imports
   - Routers: Auto-loaded from `src/routers/*/index.js` modules (agent, conversation, file, platform, model, etc.)
   - Models: Sequelize ORM in `src/models/` with SQLite (see `STORAGE_PATH` in `.env`)

2. **Frontend (Vue 3 + Vite + Ant Design Vue)**
   - Dev server: port 5005 (`frontend/package.json`)
   - Router: Hash-based (`frontend/src/router/index.js`)
   - Main views: `lemon` (chat), `setting`, `share`, `auth`
   - Backend URL: `VITE_SERVICE_URL` (default: `http://127.0.0.1:3000`)

3. **Agent System (`src/agent/AgenticAgent.js`)**
   - **Planning modes**: `base`, `server`, `local` (see `src/agent/planning/index.js`)
   - **Execution**: Code-Act loop in `src/agent/code-act/code-act.js` with retry logic
   - **Task tracking**: Markdown-based `TaskManager.js` for execution logs
   - **Memory**: Conversation history via `conversationHistoryUtils.js`
   - **Reflection**: Self-improvement through `src/agent/reflection/`

4. **Runtime Abstraction**
   - **`RUNTIME_TYPE`** env var determines execution mode:
     - `docker`: Production sandbox (`src/runtime/DockerRuntime.js`) via Aliyun ECI
     - `local-docker`: Dev sandbox (`src/runtime/DockerRuntime.local.js`) using local Docker
     - `local`: Direct execution (`src/runtime/LocalRuntime.js`) - dev only
   - All runtimes implement: `connect_container()`, `do_action(action)`, `read_file()`, `write_code()`

5. **Tool System**
   - Tools auto-loaded from `src/tools/*.js` (excluding `browser_use`)
   - Interface: `{ name, description, params, memorized?, execute(args, uuid, context) }`
   - Type definitions: `types/Tool.d.ts` (JSDoc compatible)
   - MCP tools: `src/mcp/tool.js` for Model Context Protocol integration

6. **LLM Completion Layer (`src/completion/`)**
   - Factory: `createLLMInstance(type, onTokenStream)` supports azure, openai, ollama, qwen, etc.
   - SSE streaming: Custom stream parser with `\n\n` delimiter
   - Chat completion: `src/agent/chat-completion/index.js` with token calculation

## Critical Workflows

### Development Commands
```bash
# Install dependencies (uses pnpm)
make init                    # Full setup: backend + frontend + DB tables
npm install                  # Backend only
cd frontend && npm install   # Frontend only

# Development
make run                     # Start both backend (port 3000) + frontend (port 5005)
npm run dev                  # Backend with nodemon
cd frontend && npm run dev   # Frontend dev server

# Database
make init-tables             # Run src/models/sync.js for Sequelize sync
node src/models/sync.js      # Manual table initialization

# Testing
npm test                     # Mocha tests in test/**/*.test.js

# Docker
make build-runtime-sandbox   # Build hexdolemonai/lemon-runtime-sandbox
make build-app              # Build hexdolemonai/lemon (app container)
```

### Electron Desktop Workflow
- **Bootstrap**: `main.js` → dynamic import `electron-store` → init Docker setup service → spawn backend (`bin/www`) → create window
- **Docker setup**: `dockerSetupService.js` checks `DOCKER_SETUP_DONE_KEY` in store, shows setup UI if needed
- **Deep links**: Handle `lemonai://` protocol for OAuth callbacks (`auth`, `pay-result`)
- **Multi-instance prevention**: `app.requestSingleInstanceLock()` pattern

### Agent Execution Pattern
```javascript
// 1. Create agent with runtime context
const agent = new AgenticAgent({
  conversation_id,
  user_id,
  onTokenStream,
  runtime_type: 'local-docker',
  mcp_server_ids: [],
  planning_mode: 'base'
});

// 2. Run with goal
await agent.run(goal); // Calls: auto-reply → planning → code-act loop → reflection → summary

// 3. Code-Act cycle (src/agent/code-act/code-act.js)
// - LLM generates action (thought + tool call)
// - Runtime executes via do_action()
// - Result stored in memory
// - Repeat until task complete or max iterations (25)
```

### Editor AI Modification Flow
- **Entry**: `src/editor/coding.js` 
- **Template**: `src/editor/template.txt` for HTML element editing prompts
- **Execution**: `src/editor/execute.js` resolves actions like `replace`, `insert`, `delete`
- **Retry**: `withLLMRetry()` wrapper for format correction (3 attempts with 1.5s delay)
- **Versioning**: `createAIVersion()` tracks changes in FileVersion model

## Project-Specific Conventions

### Module Resolution
- **Backend**: Uses `module-alias` with `@src/` mapping to `src/` (see `package.json._moduleAliases`)
- **Frontend**: Vite alias `@/` → `frontend/src/` (see `vite.config.js`)
- **Imports**: Always use aliases: `require('@src/models/User')` not `require('../models/User')`

### Message Streaming Protocol
```javascript
// Format: src/utils/message.js
Message.format({
  uuid,              // Unique action ID
  action_type,       // e.g., 'code_execute', 'web_search', 'chat'
  status,           // 'pending', 'success', 'error'
  content,          // Result text
  json,             // Structured data
  task_id,          // Planning task ID
  meta_content,     // Metadata
  filepath          // File operation path
});
```

### File Organization Patterns
- **Workspace**: `WORKSPACE_DIR/Conversation_<first6chars>/` per conversation
- **File versioning**: `src/models/FileVersion.js` tracks AI edits with `version_name` like `V_ai_1`
- **Static files**: Docker runtime creates nginx configs for HTML preview

### Error Handling
- **LLM retries**: Use `src/editor/withRetry.js` pattern for format errors
- **Code-Act retries**: `code-act.js` has 3-attempt retry with reflection on consecutive failures
- **Tool execution**: Return `{ status: 'error', content: errorMessage }` format

### Database Models
- **Sequelize**: All models in `src/models/`, auto-sync via `sync.js`
- **Key models**: Conversation, Message, File, FileVersion, Agent, Model, Platform, User, McpServer
- **Storage**: SQLite at `data/database.sqlite` (or `STORAGE_PATH` env)

### Testing Strategy
- **Framework**: Mocha + Chai + Supertest
- **Location**: `test/api/<module>/<module>.test.js`
- **Pattern**: `describe()` groups, `it()` tests, `request(server).get('/path')` for API tests
- **Example**: See `test/api/platform/platform.test.js` for CRUD patterns

### Environment Configuration
- **Backend**: `.env` at root (see `API_README.md` for vars like `STORAGE_PATH`, `WORKSPACE_DIR`, `RUNTIME_TYPE`)
- **Frontend**: `frontend/.env` with `VITE_SERVICE_URL`, `VITE_PORT` (see `WEB_README.md`)
- **Electron**: `.env` loaded from `process.resourcesPath` in production

## Integration Points

### Docker Runtime Communication
- **Local Docker**: Direct socket via `/var/run/docker.sock` mount
- **Port allocation**: Dynamic ranges for execution (30000-39999), VSCode (40000-49999), apps (50000-59999)
- **Container naming**: `user-<user_id>-lemon-runtime-sandbox`
- **Image**: `hexdolemonai/lemon-runtime-sandbox:latest`

### MCP (Model Context Protocol)
- **Server management**: `src/models/McpServer.js` stores configs
- **Tool wrapper**: `src/mcp/tool.js` exposes MCP tools as standard Tool interface
- **Usage**: Pass `mcp_server_ids` to AgenticAgent constructor

### Browser Automation
- **Python service**: `browser_server/` with Playwright integration
- **Bridge**: `src/tools/browser_use.js` (currently in ignored list)

## Common Patterns

### Adding a New Router Module
1. Create `src/routers/<module>/index.js` with router instance
2. Add `<module>` to `modules` array in `src/routers/index.js`
3. Use `@src/models/<Model>` for data access
4. Export routes with `router.routes()` pattern

### Creating a New Tool
```javascript
// src/tools/my_tool.js
module.exports = {
  name: "my_tool",
  description: "Clear description for LLM",
  params: {
    type: "object",
    properties: {
      arg1: { type: "string", description: "..." }
    },
    required: ["arg1"]
  },
  memorized: true, // Include output in agent memory
  async execute(args, uuid, context) {
    // Implementation
    return {
      uuid,
      status: 'success',
      content: result,
      meta: { action_type: 'my_tool' }
    };
  }
};
```

### Adding an LLM Provider
1. Create `src/completion/llm.<provider>.js` extending base pattern
2. Implement streaming SSE handler (see `llm.azure.openai.js`)
3. Add to `map` in `src/completion/index.js`
4. Update model configs in `public/default_data/`

---

**When making changes**: Always check `RUNTIME_TYPE` context, use module aliases, follow the Tool interface contract, and maintain SSE streaming format for real-time updates.
