# Codex Bridge MCP Server in D

## Quick Start (TL;DR)

```bash
# 1. Install D toolchain
brew install ldc dub

# 2. Create project (after we implement)
cd ~/projects/codex-bridge
dub build --compiler=ldc2

# 3. Register with Claude Code
claude mcp add --transport stdio codex-bridge -- ~/projects/codex-bridge/bin/codex-bridge

# 4. Use in conversation
"Use consensus to analyze this payment reconciliation bug"
```

---

## Goal

Build an MCP server in D that bridges Claude Code to OpenAI's Codex CLI, enabling consensus reasoning where both models analyze problems independently before synthesizing conclusions.

## Design Decisions

| Decision | Choice |
|----------|--------|
| **Project location** | Standalone repo (not inside orthlyweb) |
| **Invocation mode** | Explicit only — user requests "use consensus" or similar |
| **D toolchain** | Needs setup (included below) |

---

## D Toolchain Setup (macOS)

### Option A: Homebrew (Recommended)

```bash
# Install DMD (reference compiler) and DUB (package manager)
brew install dmd dub

# Verify installation
dmd --version
dub --version
```

### Option B: LDC (LLVM-based, better optimizations)

```bash
# Install LDC (produces faster binaries)
brew install ldc dub

# Verify
ldc2 --version
dub --version
```

### Option C: Official Installer

```bash
# Download from dlang.org
curl -fsS https://dlang.org/install.sh | bash -s ldc

# Activate in current shell
source ~/dlang/ldc-*/activate
```

### Recommended: LDC for Release Builds

For this project, I recommend:
- **DMD** for development (faster compilation)
- **LDC** for release builds (better runtime performance)

DUB can switch compilers via flag:
```bash
dub build                    # Uses default (dmd)
dub build --compiler=ldc2    # Uses LDC
```

---

## Research Summary

### Codex CLI Interface

| Aspect | Details |
|--------|---------|
| **Command** | `codex exec --json "prompt"` |
| **Reasoning Effort** | `-c model_reasoning_effort=<level>` where level is `minimal`, `low`, `medium`, `high`, `xhigh` |
| **xhigh availability** | Only on `gpt-5.1-codex-max` and `gpt-5.2` |
| **Output format** | JSONL stream with event types: `thread.started`, `turn.completed`, `item.completed`, `error` |
| **Item types** | `agent_message`, `reasoning`, `command_execution`, `file_change` |
| **Final output** | `-o <file>` writes final assistant message to file |
| **Full auto mode** | `--full-auto` permits file modifications |

**Sources:** [Codex CLI Reference](https://developers.openai.com/codex/cli/reference/), [Codex Config](https://github.com/openai/codex/blob/main/docs/config.md)

### MCP Protocol Requirements

| Requirement | Implementation |
|-------------|----------------|
| **Transport** | stdio (stdin/stdout) |
| **Message format** | JSON-RPC 2.0 |
| **Protocol version** | `2025-11-25` |
| **Required methods** | `initialize`, `tools/list`, `tools/call` |
| **Tool result format** | `{content: [{type: "text", text: "..."}]}` |

**Sources:** [MCP Specification](https://modelcontextprotocol.io/specification/2025-11-25)

### D Language Stack

| Component | Library |
|-----------|---------|
| **Process spawning** | `std.process.pipeProcess` |
| **JSON handling** | `std.json` (built-in) |
| **Package manager** | `dub` |
| **Async I/O** | `vibe.d` (optional, may be overkill for stdio) |

**Sources:** [std.process](https://dlang.org/phobos/std_process.html), [std.json](https://dlang.org/phobos/std_json.html), [DUB](https://dub.pm/)

### Claude Code MCP Registration

```json
// .mcp.json or ~/.claude.json
{
  "mcpServers": {
    "codex-bridge": {
      "type": "stdio",
      "command": "/path/to/codex-bridge",
      "args": [],
      "env": {
        "CODEX_API_KEY": "${CODEX_API_KEY}"
      }
    }
  }
}
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Claude Code                               │
│  (calls MCP tool: codex_analyze)                                │
└─────────────────────┬───────────────────────────────────────────┘
                      │ JSON-RPC over stdio
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Codex Bridge (D)                               │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │ MCP Server  │  │   Codex     │  │   Result Parser &       │  │
│  │  (stdio)    │──│  Invoker    │──│   Structurer            │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
│         │                │                      │                │
│         │                ▼                      │                │
│         │       ┌─────────────┐                 │                │
│         │       │ pipeProcess │                 │                │
│         │       │ (codex exec)│                 │                │
│         │       └─────────────┘                 │                │
└─────────┴───────────────┴───────────────────────┴───────────────┘
                          │
                          ▼
                    Codex CLI
                 (xhigh reasoning)
```

---

## Project Structure

```
codex-bridge/
├── dub.sdl                    # DUB package configuration
├── source/
│   ├── app.d                  # Entry point, main loop
│   ├── mcp/
│   │   ├── protocol.d         # JSON-RPC message types
│   │   ├── server.d           # MCP server implementation
│   │   └── transport.d        # stdio transport layer
│   ├── codex/
│   │   ├── invoker.d          # Codex CLI invocation
│   │   ├── parser.d           # JSONL output parser
│   │   └── types.d            # Codex response types
│   └── tools/
│       └── analyze.d          # codex_analyze tool implementation
└── test/
    ├── mcp_test.d             # MCP protocol tests
    └── codex_test.d           # Codex invocation tests
```

---

## Implementation Plan

### Phase 1: Project Setup

1. **Initialize DUB project**
   ```bash
   mkdir codex-bridge && cd codex-bridge
   dub init --format sdl
   ```

2. **Configure `dub.sdl`**
   ```sdl
   name "codex-bridge"
   description "MCP server bridging Claude Code to Codex CLI"

   dependency "mir-ion" version="~>2.2"  # Optional: faster JSON

   targetType "executable"
   targetPath "bin"

   configuration "release" {
       dflags "-O" "-release"
   }
   ```

### Phase 2: MCP Protocol Layer

**File: `source/mcp/protocol.d`**

Implement JSON-RPC 2.0 message types:
- `JsonRpcRequest` struct with `jsonrpc`, `id`, `method`, `params`
- `JsonRpcResponse` struct with `jsonrpc`, `id`, `result`/`error`
- `JsonRpcError` struct with `code`, `message`, `data`

**File: `source/mcp/transport.d`**

Implement stdio transport:
- Read lines from stdin
- Parse JSON-RPC messages
- Write JSON responses to stdout
- Handle buffering correctly

**File: `source/mcp/server.d`**

Implement MCP server logic:
- Handle `initialize` → return capabilities
- Handle `tools/list` → return tool definitions
- Handle `tools/call` → dispatch to tool handlers
- Route unknown methods to error response

### Phase 3: Codex Integration

**File: `source/codex/types.d`**

Define response structures:
```d
struct CodexAnalysisResult {
    string reasoning;
    string conclusion;
    Hypothesis[] hypotheses;
    string[] suggestedInvestigations;
    ProposedFix[] proposedFixes;
}

struct Hypothesis {
    string description;
    double confidence;
    string[] evidence;
}

struct ProposedFix {
    string file;
    string description;
    string diff;
}
```

**File: `source/codex/invoker.d`**

Implement Codex CLI invocation:
```d
auto invokeCodex(string prompt, string effort, string[] contextFiles) {
    // Build command: codex exec --json -c model_reasoning_effort=xhigh ...
    auto pipes = pipeProcess(["codex", "exec", "--json",
                              "-c", "model_reasoning_effort=" ~ effort,
                              prompt]);

    // Read JSONL output
    string[] events;
    foreach (line; pipes.stdout.byLine) {
        events ~= line.idup;
    }

    // Wait for completion
    wait(pipes.pid);

    return events;
}
```

**File: `source/codex/parser.d`**

Parse JSONL events:
- Filter for `item.completed` events with `agent_message` type
- Extract reasoning traces from `reasoning` items
- Build structured `CodexAnalysisResult`

### Phase 4: Tool Implementation

**File: `source/tools/analyze.d`**

The `codex_analyze` tool:

```d
struct AnalyzeParams {
    string prompt;
    string effort;           // "high" | "xhigh"
    string[] contextFiles;
    string[] focusAreas;
    string mode;             // "debug" | "implement" | "review" | "architecture"
}

JSONValue handleAnalyze(AnalyzeParams params) {
    // 1. Build context bundle from files
    string context = buildContextBundle(params.contextFiles);

    // 2. Construct tailored prompt based on mode
    string fullPrompt = buildPrompt(params.mode, params.focusAreas,
                                    params.prompt, context);

    // 3. Invoke Codex
    auto events = invokeCodex(fullPrompt, params.effort);

    // 4. Parse and structure response
    auto result = parseCodexOutput(events);

    // 5. Return as MCP tool result
    return JSONValue([
        "content": JSONValue([
            JSONValue([
                "type": JSONValue("text"),
                "text": JSONValue(result.toJson())
            ])
        ])
    ]);
}
```

### Phase 5: Main Loop

**File: `source/app.d`**

```d
void main() {
    auto server = new McpServer();
    server.registerTool("codex_analyze", &handleAnalyze);

    // Main loop: read stdin, process, write stdout
    while (!stdin.eof) {
        auto line = stdin.readln();
        if (line.empty) continue;

        auto request = parseJsonRpc(line);
        auto response = server.handleRequest(request);
        writeln(response.toJson());
        stdout.flush();
    }
}
```

### Phase 6: Testing

1. **Unit tests for JSON-RPC parsing**
2. **Integration test with mock Codex output**
3. **End-to-end test with actual Codex CLI**

### Phase 7: Claude Code Integration

1. **Build release binary**
   ```bash
   dub build --config=release
   ```

2. **Register with Claude Code**
   ```bash
   claude mcp add --transport stdio codex-bridge -- \
       /path/to/codex-bridge/bin/codex-bridge
   ```

3. **Test invocation**
   ```
   > /mcp  # Check server is connected
   > Ask Claude to use codex_analyze tool
   ```

---

## Tool Schema

```json
{
  "name": "codex_analyze",
  "description": "Deep analysis via OpenAI Codex with configurable reasoning effort. Use for complex bugs, architectural decisions, or when you want a second opinion with high reasoning depth.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "prompt": {
        "type": "string",
        "description": "The analysis task or question"
      },
      "effort": {
        "type": "string",
        "enum": ["high", "xhigh"],
        "description": "Reasoning effort level. 'xhigh' for maximum depth."
      },
      "contextFiles": {
        "type": "array",
        "items": {"type": "string"},
        "description": "Paths to files to include as context"
      },
      "focusAreas": {
        "type": "array",
        "items": {"type": "string"},
        "description": "Specific areas to focus on (e.g., 'race conditions', 'error handling')"
      },
      "mode": {
        "type": "string",
        "enum": ["debug", "implement", "review", "architecture"],
        "description": "Analysis mode determining prompt structure"
      }
    },
    "required": ["prompt", "effort", "mode"]
  }
}
```

---

## Consensus Workflow Integration

**Invocation: Explicit only.** You'll trigger consensus analysis with phrases like:
- "use consensus for this"
- "get a second opinion from Codex"
- "deep analyze with both models"

Once triggered, the orchestration pattern:

1. **You request consensus analysis**
2. **I spawn parallel analysis:**
   - Call `codex_analyze` with `effort: "xhigh"`
   - Simultaneously perform my own investigation
3. **I receive Codex result**
4. **Cross-check phase:**
   - Identify agreements (high confidence)
   - Identify disagreements (flag for deeper investigation)
   - Note unique insights from each model
5. **Synthesized output** — you get both perspectives + combined conclusion

---

## Open Considerations

### Context Size Management
- 200k token budget for Codex
- If context exceeds budget, chunk into multiple calls with summarized context
- Consider implementing a context estimator (rough token count: chars / 4)

### Disagreement Resolution
- Default: Flag disagreements + attempt verification via codebase evidence
- After 2-3 rounds of verification without resolution → bubble to user
- Track confidence levels from both models

### Error Handling
- Codex CLI unavailable → clear error message
- API rate limits → exponential backoff
- Malformed output → fallback to raw text response

---

## Files to Create

Standalone repo at `~/projects/codex-bridge/` (or location of your choice):

| File | Purpose |
|------|---------|
| `dub.sdl` | Package configuration |
| `source/app.d` | Entry point |
| `source/mcp/protocol.d` | JSON-RPC types |
| `source/mcp/transport.d` | stdio I/O |
| `source/mcp/server.d` | MCP handler |
| `source/codex/types.d` | Result types |
| `source/codex/invoker.d` | CLI invocation |
| `source/codex/parser.d` | JSONL parsing |
| `source/tools/analyze.d` | Tool implementation |

**Total:** ~9 source files, ~600-800 lines of D code

---

## References

- [Codex CLI Reference](https://developers.openai.com/codex/cli/reference/)
- [Codex Config](https://github.com/openai/codex/blob/main/docs/config.md)
- [MCP Specification 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25)
- [D std.process](https://dlang.org/phobos/std_process.html)
- [D std.json](https://dlang.org/phobos/std_json.html)
- [DUB Package Manager](https://dub.pm/)
