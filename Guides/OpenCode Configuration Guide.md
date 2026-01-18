---
tags:
  - opencode
  - ai
  - config
  - dev-tools
  - guide
date: 2026-01-14
---

# OpenCode Configuration Guide

Complete step-by-step guide for configuring OpenCode: from basic setup to advanced configuration with MCP servers, custom agents, commands, and permissions.

## Installation

```bash
# Via install script (recommended)
curl -fsSL https://opencode.ai/install | bash

# Via npm
npm install -g opencode-ai

# Via Homebrew (macOS/Linux)
brew install anomalyco/tap/opencode

# Via Chocolatey (Windows)
choco install opencode

# Via Scoop (Windows)
scoop bucket add extras
scoop install extras/opencode
```

## Initial Setup

### 1. Connect Provider

```bash
# Launch TUI
opencode

# Inside interface, run:
/connect
```

**Recommended:** Use **OpenCode Zen** for beginners - it's a curated list of verified models.

Alternatively, select from other providers like Anthropic, OpenAI, Google, etc.

### 2. Initialize Project

```bash
cd /path/to/project
opencode
/init
```

The `/init` command creates an `AGENTS.md` file in project root with context about structure and code patterns.

**Important:** Commit `AGENTS.md` to Git - it helps OpenCode understand your project.

## Configuration Structure

OpenCode uses a **layered configuration system** with clear merge precedence (later overrides earlier):

### Precedence Order

1. **Remote config** - Organizational defaults from `.well-known/opencode`
2. **Global config** - User preferences at `~/.config/opencode/opencode.json`
3. **Custom config** - Via `OPENCODE_CONFIG` environment variable
4. **Project config** - Project-specific at `<project>/opencode.json`
5. **`.opencode` directories** - Agents, commands, plugins
6. **Inline config** - Runtime override via `OPENCODE_CONFIG_CONTENT`

**Key principle:** Configs are **merged**, not replaced. Settings from all layers combine, with later configs only overriding conflicting keys.

**Example:** If global config sets `theme: "opencode"` and `autoupdate: true`, and project config sets `model: "anthropic/claude-sonnet-4-5"`, the final configuration includes **all three** settings.

### Directory Structure

```
~/.config/opencode/              # Global user config
├── opencode.json                # Main global settings
├── AGENTS.md                    # Personal instructions (not committed)
├── agent/                       # Custom agent definitions
│   ├── code-reviewer.md
│   └── docs-writer.md
├── command/                     # Custom commands
│   └── test.md
└── plugin/                      # Custom plugins (optional)

<project-root>/                  # Project-specific
├── opencode.json                # Project config (COMMIT TO GIT!)
├── AGENTS.md                    # Project rules (COMMIT TO GIT!)
└── .opencode/                   # Project-specific extensions
    ├── agent/
    ├── command/
    └── plugin/
```

## Step 1: Global Configuration

**File:** `~/.config/opencode/opencode.json`

This stores user-wide settings that apply across all projects.

### Comprehensive Global Config Example

```json
{
  "$schema": "https://opencode.ai/config.json",
  
  // Model Configuration
  "model": "anthropic/claude-sonnet-4-5",
  "small_model": "anthropic/claude-haiku-4-5",
  
  // Provider Configuration
  "provider": {
    "anthropic": {
      "options": {
        "timeout": 600000,        // 10 minutes
        "setCacheKey": true       // Always set cache key
      }
    }
  },
  
  // UI Settings
  "theme": "opencode",
  "autoupdate": true,              // or "notify" or false
  "share": "manual",               // "manual", "auto", or "disabled"
  
  // TUI-specific settings
  "tui": {
    "scroll_speed": 3,
    "scroll_acceleration": {
      "enabled": true              // macOS-style acceleration
    },
    "diff_style": "auto"           // "auto" or "stacked"
  },
  
  // Tool Configuration (global defaults)
  "tools": {
    "write": true,
    "edit": true,
    "bash": true,
    "webfetch": true
  },
  
  // Permission Settings
  "permission": {
    "edit": "allow",               // "allow", "ask", "deny"
    "bash": "allow",
    "webfetch": "allow"
  },
  
  // Context Management
  "compaction": {
    "auto": true,                  // Auto-compact when context full
    "prune": true                  // Remove old tool outputs
  },
  
  // File Watcher (ignore patterns)
  "watcher": {
    "ignore": [
      "node_modules/**",
      "dist/**",
      ".git/**",
      "build/**",
      ".next/**",
      "venv/**",
      "__pycache__/**"
    ]
  },
  
  // Default agent (must be primary, not subagent)
  "default_agent": "build"
}
```

### Key Settings Explained

- **`model`**: Main model for all tasks (Claude Sonnet 4.5)
- **`small_model`**: Cheaper model for lightweight tasks (title generation, etc.)
- **`provider.*.options.timeout`**: Request timeout in milliseconds
- **`provider.*.options.setCacheKey`**: Ensures cache keys for provider
- **`share`**: Control conversation sharing (manual/auto/disabled)
- **`tools`**: Global enable/disable tools
- **`permission`**: Control tool permissions (allow/ask/deny)
- **`compaction.auto`**: Auto-compact when context fills up
- **`default_agent`**: Which primary agent is used by default

## Step 2: Global AGENTS.md

**File:** `~/.config/opencode/AGENTS.md`

Contains **personal instructions** applied to all OpenCode sessions. **NOT committed to Git** (unlike project-specific AGENTS.md).

### Example Global AGENTS.md

```markdown
# My Personal OpenCode Rules

## Communication Style
- Be concise and technical in responses
- Avoid unnecessary pleasantries
- Focus on practical solutions

## Code Preferences
- Use TypeScript for all new JavaScript code
- Prefer functional programming patterns
- Write clear, self-documenting code with minimal comments
- Use descriptive variable names

## Testing Standards
- Write tests for new features
- Prefer integration tests over unit tests
- Use meaningful test descriptions

## Git Workflow
- Write clear, imperative commit messages
- Reference issue numbers when applicable
- Ask before force pushing

## Tool Usage
- Use MCP tools when searching documentation
- Check existing code patterns before implementing new solutions
```

### Purpose

- **Global AGENTS.md** = Personal preferences (not shared with team)
- **Project AGENTS.md** = Project rules (commit to Git, shared with team)

## Step 3: Configure Built-in Agents

OpenCode has 4 built-in agents that can be customized.

### Agent Types

**Primary agents** - Main assistants (switch via **Tab** key):
- **Build** - Full tool access, default agent
- **Plan** - Planning mode without code changes (permissions on `ask`)

**Subagents** - Helper agents (invoke via **`@mention`** or automatically):
- **General** - General tasks and research, multi-step tasks
- **Explore** - Fast codebase exploration and searching

### Customizing Built-in Agents

Add to your `opencode.json`:

```json
{
  "agent": {
    "build": {
      "mode": "primary",
      "description": "Full development mode with all tools enabled",
      "model": "anthropic/claude-sonnet-4-5",
      "temperature": 0.3,          // Balanced creativity
      "tools": {
        "write": true,
        "edit": true,
        "bash": true
      }
    },
    
    "plan": {
      "mode": "primary",
      "description": "Analysis and planning mode without making changes",
      "model": "anthropic/claude-haiku-4-5",  // Fast cheap model
      "temperature": 0.1,          // Very focused
      "permission": {
        "edit": "ask",             // Requires confirmation
        "bash": "ask",
        "write": "ask"
      }
    },
    
    "general": {
      "mode": "subagent",
      "description": "General-purpose research and multi-step tasks",
      "model": "anthropic/claude-sonnet-4-5"
    },
    
    "explore": {
      "mode": "subagent",
      "description": "Fast codebase exploration and searching",
      "model": "anthropic/claude-haiku-4-5"
    }
  },
  
  "default_agent": "build"
}
```

### Temperature Guidelines

- **0.0-0.2**: Very focused (code analysis, planning)
- **0.3-0.5**: Balanced (general development)
- **0.6-1.0**: Creative (brainstorming, exploration)

## Step 4: Create Custom Agents

### Method 1: Interactive Command

```bash
opencode agent create
```

This will:
1. Ask where to save (global or project-specific)
2. Request agent description
3. Let you select available tools
4. Generate markdown file with config

### Method 2: JSON Config

Add to `~/.config/opencode/opencode.json`:

```json
{
  "agent": {
    "code-reviewer": {
      "description": "Reviews code for best practices and potential issues",
      "mode": "subagent",
      "model": "anthropic/claude-sonnet-4-5",
      "temperature": 0.1,
      "prompt": "{file:./prompts/code-review.txt}",
      "tools": {
        "write": false,
        "edit": false,
        "bash": false
      },
      "permission": {
        "edit": "deny",
        "bash": "ask"
      }
    }
  }
}
```

### Method 3: Markdown Files

**Locations:**
- Global: `~/.config/opencode/agent/`
- Project: `.opencode/agent/`

**Example:** `~/.config/opencode/agent/code-reviewer.md`

```markdown
---
description: Reviews code for best practices and potential issues
mode: subagent
model: anthropic/claude-sonnet-4-5
temperature: 0.1
tools:
  write: false
  edit: false
  bash: false
---

You are a code reviewer. Focus on:

- Code quality and best practices
- Potential bugs and edge cases
- Performance implications
- Security considerations
- Maintainability

Provide constructive feedback without making direct changes.
```

**Example:** `~/.config/opencode/agent/docs-writer.md`

```markdown
---
description: Writes and maintains project documentation
mode: subagent
tools:
  bash: false
---

You are a technical writer. Create clear, comprehensive documentation.

Focus on:
- Clear explanations
- Proper structure
- Code examples
- User-friendly language
```

**Example:** `~/.config/opencode/agent/security-auditor.md`

```markdown
---
description: Performs security audits and identifies vulnerabilities
mode: subagent
temperature: 0.1
tools:
  write: false
  edit: false
permission:
  bash: deny
---

You are a security expert. Focus on identifying potential security issues.

Look for:
- Input validation vulnerabilities
- Authentication and authorization flaws
- Data exposure risks
- Dependency vulnerabilities
- Configuration security issues
```

### Agent Options Reference

- **`description`** - Agent description (REQUIRED)
- **`mode`** - `"primary"`, `"subagent"`, or `"all"` (default: `"all"`)
- **`model`** - Override model for this agent
- **`temperature`** - Creativity (0.0-1.0), default depends on model
- **`maxSteps`** - Limit agentic iterations (default: unlimited)
- **`prompt`** - Custom system prompt (can use `{file:path}`)
- **`tools`** - Available tools (object with boolean values)
- **`permission`** - Tool permissions (allow/ask/deny)
- **`hidden`** - Hide subagent from `@` autocomplete menu (default: false)
- **`disable`** - Disable agent (default: false)

**Additional options** are passed directly to provider as model options (e.g., `reasoningEffort` for OpenAI).

## Step 5: Agent Permissions

Control tool access through permission system.

### Basic Permissions

```json
{
  "agent": {
    "build": {
      "permission": {
        "edit": "ask",           // Requires confirmation
        "bash": "allow",         // Allow without confirmation
        "webfetch": "deny"       // Completely deny
      }
    }
  }
}
```

**Values:**
- **`"allow"`** - Allow without confirmation
- **`"ask"`** - Require user approval before execution
- **`"deny"`** - Completely forbidden (tool unavailable)

### Bash Command Permissions (with glob patterns)

```json
{
  "agent": {
    "build": {
      "permission": {
        "bash": {
          "*": "ask",              // All commands require confirmation
          "git status": "allow",   // Except git status
          "git log*": "allow",     // And git log with any args
          "git push": "deny"       // git push is forbidden
        }
      }
    }
  }
}
```

**Important:** Last matching rule takes precedence. Put `*` wildcard **first**, specific rules **after**.

### Task Permissions (controlling subagent invocation)

```json
{
  "agent": {
    "orchestrator": {
      "mode": "primary",
      "permission": {
        "task": {
          "*": "deny",                    // Deny all subagents
          "orchestrator-*": "allow",      // Except orchestrator-*
          "code-reviewer": "ask"          // code-reviewer requires confirmation
        }
      }
    }
  }
}
```

**Behavior:**
- `"deny"` - Subagent removed from Task tool description (model won't see it)
- `"ask"` - Requires user confirmation
- `"allow"` - Allowed without confirmation

**Note:** User can always invoke any subagent via `@mention`, even if task permissions deny.

## Step 6: MCP Servers

Model Context Protocol (MCP) adds external tools to OpenCode.

**⚠️ WARNING:** MCP servers add context to every request! Be selective - use only necessary servers. GitHub MCP, for example, can significantly increase token usage.

### MCP Server Types

1. **Local MCP** - Run locally via command
2. **Remote MCP** - HTTP endpoints with tools
3. **OAuth MCP** - Remote servers with OAuth authentication

### Enable/Disable MCP Servers

**Global enable/disable:**

```json
{
  "mcp": {
    "my-mcp": {
      "type": "local",
      "command": ["npx", "-y", "my-mcp-server"],
      "enabled": true              // or false to disable
    }
  }
}
```

**Override remote defaults:**

Organizations can provide MCP via `.well-known/opencode`. To override:

```json
{
  "mcp": {
    "jira": {
      "type": "remote",
      "url": "https://jira.example.com/mcp",
      "enabled": true              // Override organizational default
    }
  }
}
```

### Local MCP Server

Local MCP runs as a process via command.

**Options:**

| Option | Type | Required | Description |
|--------|------|----------|-------------|
| `type` | String | ✓ | `"local"` |
| `command` | Array | ✓ | Command and args to run |
| `environment` | Object | | Environment variables |
| `enabled` | Boolean | | Enable/disable on startup |
| `timeout` | Number | | Timeout in ms (default: 5000) |

**Example:**

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "my-local-mcp": {
      "type": "local",
      "command": ["npx", "-y", "@modelcontextprotocol/server-everything"],
      "enabled": true,
      "environment": {
        "MY_ENV_VAR": "my_env_var_value",
        "NODE_ENV": "production"
      },
      "timeout": 10000
    }
  }
}
```

### Remote MCP Server

Remote MCP - HTTP endpoint with tools.

**Options:**

| Option | Type | Required | Description |
|--------|------|----------|-------------|
| `type` | String | ✓ | `"remote"` |
| `url` | String | ✓ | URL of remote MCP server |
| `enabled` | Boolean | | Enable/disable on startup |
| `headers` | Object | | HTTP headers for request |
| `oauth` | Object/false | | OAuth config or `false` to disable |
| `timeout` | Number | | Timeout in ms (default: 5000) |

**Example:**

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "my-remote-mcp": {
      "type": "remote",
      "url": "https://my-mcp-server.com",
      "enabled": true,
      "headers": {
        "Authorization": "Bearer {env:MY_API_KEY}",
        "X-Custom-Header": "value"
      },
      "timeout": 10000
    }
  }
}
```

### MCP with OAuth

OpenCode automatically handles OAuth for remote MCP servers via **Dynamic Client Registration (RFC 7591)**.

#### Automatic OAuth

For most OAuth-enabled servers, minimal config needed:

```json
{
  "mcp": {
    "my-oauth-server": {
      "type": "remote",
      "url": "https://mcp.example.com/mcp"
      // OAuth auto-detected on 401 response
    }
  }
}
```

If server requires authentication, OpenCode auto-launches OAuth flow on first use.

#### Pre-registered OAuth (with client credentials)

If you have client credentials from provider:

```json
{
  "mcp": {
    "my-oauth-server": {
      "type": "remote",
      "url": "https://mcp.example.com/mcp",
      "oauth": {
        "clientId": "{env:MY_MCP_CLIENT_ID}",
        "clientSecret": "{env:MY_MCP_CLIENT_SECRET}",
        "scope": "tools:read tools:execute"
      }
    }
  }
}
```

#### Disable OAuth

For servers with API keys instead of OAuth:

```json
{
  "mcp": {
    "my-api-key-server": {
      "type": "remote",
      "url": "https://mcp.example.com/mcp",
      "oauth": false,                              // Disable OAuth auto-detection
      "headers": {
        "Authorization": "Bearer {env:MY_API_KEY}"
      }
    }
  }
}
```

#### OAuth Commands

```bash
# Authenticate with MCP server
opencode mcp auth my-oauth-server

# List all MCP and their auth status
opencode mcp list
opencode mcp auth list

# Remove stored credentials
opencode mcp logout my-oauth-server

# Debug OAuth connection
opencode mcp debug my-oauth-server
```

**Stored tokens:** `~/.local/share/opencode/mcp-auth.json`

### Popular MCP Server Examples

#### Context7 - Documentation Search

Search technical documentation.

```json
{
  "mcp": {
    "context7": {
      "type": "remote",
      "url": "https://mcp.context7.com/mcp",
      "enabled": true
    }
  }
}
```

With API key for higher rate-limits (optional):

```json
{
  "mcp": {
    "context7": {
      "type": "remote",
      "url": "https://mcp.context7.com/mcp",
      "headers": {
        "CONTEXT7_API_KEY": "{env:CONTEXT7_API_KEY}"
      }
    }
  }
}
```

**Usage:**

```
Configure a Cloudflare Worker script to cache JSON API responses. use context7
```

Or add to `AGENTS.md`:

```markdown
When you need to search docs, use `context7` tools.
```

#### GitHub Grep - Code Search

Search code snippets on GitHub via [grep.app](https://grep.app).

```json
{
  "mcp": {
    "gh_grep": {
      "type": "remote",
      "url": "https://mcp.grep.app",
      "enabled": true
    }
  }
}
```

**Usage:**

```
What's the right way to set a custom domain in an SST Astro component? use gh_grep
```

Or in `AGENTS.md`:

```markdown
If you are unsure how to do something, use `gh_grep` to search code examples from GitHub.
```

#### Sentry - Error Tracking

Integrate with Sentry for issues and projects.

```json
{
  "mcp": {
    "sentry": {
      "type": "remote",
      "url": "https://mcp.sentry.dev/mcp",
      "oauth": {}
    }
  }
}
```

**Authentication:**

```bash
opencode mcp auth sentry
```

**Usage:**

```
Show me the latest unresolved issues in my project. use sentry
```

### Managing MCP as Tools

MCP servers register as tools, so you can manage them via `tools` config.

#### Globally Disable MCP

```json
{
  "mcp": {
    "my-mcp-foo": {
      "type": "local",
      "command": ["bun", "x", "my-mcp-command-foo"]
    },
    "my-mcp-bar": {
      "type": "local",
      "command": ["bun", "x", "my-mcp-command-bar"]
    }
  },
  "tools": {
    "my-mcp-foo_*": false,         // Disable all tools from my-mcp-foo
    "my-mcp-bar_search": false     // Disable specific tool
  }
}
```

**Glob patterns:**
- `*` - Any number of any characters
- `?` - Exactly one character
- Other characters - Literal match

**Important:** MCP tools register with server name as prefix: `servername_toolname`

To disable **all** tools from a server: `"servername_*": false`

#### Enable MCP Only for Specific Agent

```json
{
  "mcp": {
    "gh_grep": {
      "type": "remote",
      "url": "https://mcp.grep.app",
      "enabled": true
    }
  },
  
  // Globally disable
  "tools": {
    "gh_grep_*": false
  },
  
  // Enable for specific agent
  "agent": {
    "researcher": {
      "description": "Research agent with GitHub search",
      "mode": "subagent",
      "tools": {
        "gh_grep_*": true           // Override global disable
      }
    }
  }
}
```

This allows **controlling context usage** - MCP tools available only when needed.

## Step 7: Custom Commands

Automate repetitive tasks.

### JSON Config

```json
{
  "command": {
    "test": {
      "template": "Run the full test suite with coverage report.\nFocus on failing tests and suggest fixes.",
      "description": "Run tests with coverage",
      "agent": "build",
      "model": "anthropic/claude-haiku-4-5"
    },
    "component": {
      "template": "Create a new React component named $ARGUMENTS with TypeScript.\nInclude proper typing and basic structure.",
      "description": "Create a new component"
    }
  }
}
```

### Markdown File

`~/.config/opencode/command/deploy.md`:

```markdown
---
description: Deploy application to production
agent: build
---

Deploy the application to production:
1. Run tests
2. Build the application
3. Deploy to server
4. Verify deployment
```

**Usage:**

```bash
/test
/component MyButton
/deploy
```

## Step 8: Code Formatters

Configure automatic code formatting.

```json
{
  "formatter": {
    "prettier": {
      "disabled": false
    },
    "custom-prettier": {
      "command": ["npx", "prettier", "--write", "$FILE"],
      "environment": {
        "NODE_ENV": "development"
      },
      "extensions": [".js", ".ts", ".jsx", ".tsx"]
    },
    "black": {
      "command": ["black", "$FILE"],
      "extensions": [".py"]
    }
  }
}
```

## Step 9: Environment Variables

Use environment variables in config for sensitive data.

**Environment variables:**

```json
{
  "model": "{env:OPENCODE_MODEL}",
  "provider": {
    "anthropic": {
      "options": {
        "apiKey": "{env:ANTHROPIC_API_KEY}"
      }
    }
  }
}
```

**Or files:**

```json
{
  "instructions": ["./custom-instructions.md"],
  "provider": {
    "openai": {
      "options": {
        "apiKey": "{file:~/.secrets/openai-key}"
      }
    }
  }
}
```

## Step 10: Project Configuration

When working in a project, override global settings via project config.

### Project opencode.json

**File:** `<project-root>/opencode.json`

**⚠️ IMPORTANT:** Commit to Git! This is shared with team.

```json
{
  "$schema": "https://opencode.ai/config.json",
  
  // Project-specific model (overrides global)
  "model": "anthropic/claude-sonnet-4-5",
  
  // Custom instructions specific to this project
  "instructions": [
    "CONTRIBUTING.md",
    "docs/coding-standards.md",
    ".cursor/rules/*.md"
  ],
  
  // Project-specific MCP servers
  "mcp": {
    "gh_grep": {
      "type": "remote",
      "url": "https://mcp.grep.app",
      "enabled": true              // Override global setting
    }
  },
  
  // Code formatters
  "formatter": {
    "prettier": {
      "command": ["npx", "prettier", "--write", "$FILE"],
      "extensions": [".js", ".ts", ".jsx", ".tsx", ".json"],
      "environment": {
        "NODE_ENV": "development"
      }
    },
    "black": {
      "command": ["black", "$FILE"],
      "extensions": [".py"]
    }
  },
  
  // Project-specific agents
  "agent": {
    "test-runner": {
      "description": "Run and analyze tests for this project",
      "mode": "subagent",
      "model": "anthropic/claude-haiku-4-5"
    }
  }
}
```

### Project AGENTS.md

**File:** `<project-root>/AGENTS.md`

**⚠️ IMPORTANT:** Commit to Git! Helps OpenCode understand project.

```markdown
# Project Name

This is a Next.js TypeScript project using App Router and Prisma.

## Project Structure
- `app/` - Next.js app router pages and layouts
- `components/` - Reusable React components
- `lib/` - Utility functions and helpers
- `prisma/` - Database schema and migrations
- `public/` - Static assets

## Code Standards
- Use TypeScript strict mode
- Follow Next.js 14+ conventions
- Use server components by default
- Client components only when needed (use "use client")
- All database queries through Prisma

## Testing
- Place tests next to source files with `.test.ts` extension
- Use Jest and React Testing Library
- Aim for 80% coverage on business logic

## Deployment
- Deployed to Vercel
- Database on PlanetScale
- Environment variables in `.env.local`
```

**Creating via `/init`:**

```bash
cd /path/to/project
opencode
/init
```

The `/init` command automatically:
1. Scans project structure
2. Identifies technologies and patterns
3. Generates `AGENTS.md` with context

### Project .opencode Directory

**Structure:**

```
<project-root>/.opencode/
├── agent/              # Project-specific agents
│   └── api-tester.md
├── command/            # Project-specific commands
│   └── deploy.md
└── plugin/             # Project-specific plugins
```

**Don't commit to Git if contains sensitive data!** Add to `.gitignore`:

```gitignore
.opencode/
!.opencode/agent/
!.opencode/command/
```

## Useful Commands

### CLI Commands

```bash
# Create agent interactively
opencode agent create

# List MCP servers and auth status
opencode mcp list
opencode mcp auth list

# Authenticate with MCP server
opencode mcp auth <server-name>

# Debug MCP connection
opencode mcp debug <server-name>

# Logout from MCP
opencode mcp logout <server-name>

# List available models
opencode models

# Run OpenCode server
opencode serve
opencode web
```

### TUI Commands

```bash
# Initialize project
/init

# Undo/redo changes
/undo
/redo

# Share conversation
/share

# Connect provider
/connect

# Custom commands
/test
/component MyButton
```

### Keybinds (default)

```
Tab                  # Switch between primary agents
<Leader>Right        # Cycle forward through child sessions
<Leader>Left         # Cycle backward through child sessions
@mention             # Invoke subagent (e.g., @general or @explore)
Ctrl+P               # List available actions
```

## Quick Reference

### Config Precedence (merge order)

1. Remote → 2. Global → 3. Custom → 4. Project → 5. .opencode → 6. Inline

**Key:** Configs **merge**, not replace. Later overrides only conflicting keys.

### File Locations

| Type | Global | Project |
|------|--------|---------|
| Config | `~/.config/opencode/opencode.json` | `<project>/opencode.json` |
| Rules | `~/.config/opencode/AGENTS.md` | `<project>/AGENTS.md` |
| Agents | `~/.config/opencode/agent/*.md` | `.opencode/agent/*.md` |
| Commands | `~/.config/opencode/command/*.md` | `.opencode/command/*.md` |
| Plugins | `~/.config/opencode/plugin/*.ts` | `.opencode/plugin/*.ts` |

### Agent Types

| Type | Switch Method | Examples |
|------|---------------|----------|
| Primary | Tab key | build, plan |
| Subagent | @mention or auto-invoke | general, explore |

### Permission Values

| Value | Behavior |
|-------|----------|
| `"allow"` | Allow without confirmation |
| `"ask"` | Require user approval |
| `"deny"` | Completely forbidden |

### Temperature Guidelines

| Range | Use Case |
|-------|----------|
| 0.0-0.2 | Code analysis, planning, focused work |
| 0.3-0.5 | General development, balanced |
| 0.6-1.0 | Brainstorming, creative exploration |

### Variable Substitution

| Syntax | Example | Purpose |
|--------|---------|---------|
| `{env:VAR}` | `{env:ANTHROPIC_API_KEY}` | Environment variables |
| `{file:path}` | `{file:~/.secrets/key}` | File contents |

## Best Practices

### 1. MCP Servers - Context Management

**❌ Avoid:**
- Enabling all available MCP servers
- Globally enabling heavy MCP (e.g., GitHub MCP)

**✅ Do:**
- Enable only necessary MCP servers
- Use per-agent MCP activation for context control
- Monitor token usage when adding new MCP

### 2. Agents - Specialization

**❌ Avoid:**
- Using one agent for all tasks
- Giving all agents full tool access

**✅ Do:**
- Create specialized agents for specific tasks
- Use read-only agents (code-reviewer, security-auditor)
- Plan agent for analysis, Build agent for implementation

### 3. Permissions - Security

**❌ Avoid:**
- `"allow"` for all critical operations
- Full bash access without restrictions

**✅ Do:**
- Use `"ask"` for file edits, bash commands, deployments
- Specific bash permissions: allow `git status`, deny `rm -rf`
- Read-only agents: `"deny"` for write/edit/bash

### 4. AGENTS.md - Context Sharing

**❌ Avoid:**
- Empty AGENTS.md or missing it
- Duplicating information from code

**✅ Do:**
- Always `/init` in new projects
- Commit project AGENTS.md to Git
- Global AGENTS.md for personal preferences (not committed)
- Add high-level context: structure, conventions, workflow

### 5. Temperature - Task Matching

**❌ Avoid:**
- High temperature for code analysis
- Same temperature for all agents

**✅ Do:**
- **0.1-0.2:** Code review, security audit, analysis
- **0.3-0.5:** General development, implementation
- **0.7-0.9:** Brainstorming, documentation writing

### 6. Models - Cost Optimization

**❌ Avoid:**
- Sonnet for all tasks
- Not configuring `small_model`

**✅ Do:**
- **Haiku:** Planning, simple tasks, fast iteration
- **Sonnet:** Complex implementation, architecture decisions
- Configure `small_model` for title generation and lightweight tasks
- Per-agent model override: haiku for plan, sonnet for build

### 7. Config Organization

**❌ Avoid:**
- Everything in one huge config file
- Hardcoded secrets in config

**✅ Do:**
- Global config: user preferences, providers
- Project config: project-specific overrides, formatters
- Markdown agents: `~/.config/opencode/agent/*.md`
- Environment variables: `{env:API_KEY}` or `{file:~/.secrets/key}`

### 8. Instructions - External Files

**❌ Avoid:**
- Duplicating CONTRIBUTING.md in AGENTS.md

**✅ Do:**
- Use `instructions` field to reference existing docs:
  ```json
  {
    "instructions": [
      "CONTRIBUTING.md",
      "docs/coding-standards.md"
    ]
  }
  ```

### 9. Formatters - Consistency

**❌ Avoid:**
- Manual formatting after each change

**✅ Do:**
- Configure formatters for automatic formatting:
  ```json
  {
    "formatter": {
      "prettier": {
        "command": ["npx", "prettier", "--write", "$FILE"],
        "extensions": [".js", ".ts", ".tsx"]
      }
    }
  }
  ```

### 10. Workflow Tips

- **Start with Plan mode** (Tab) for analysis before implementation
- **Use `/undo`** if result isn't what you expected
- **`@explore`** for quick codebase search
- **`@general`** for multi-step research tasks
- **`/share`** for sharing conversations with team
- **Commit opencode.json and AGENTS.md** to project repo

## Related Notes

- [[AGENTS]] - Context file for AI agents (this vault)
- [[Commands/stow git]] - Git dotfiles management with stow
- [[sync via ssh bi-directional]] - Unison bi-directional sync setup
- [[hotkeys/Obsidian]] - Obsidian hotkeys reference

## Links

- **Documentation:** https://opencode.ai/docs
- **GitHub:** https://github.com/anomalyco/opencode
- **Discord:** https://opencode.ai/discord
- **Config Schema:** https://opencode.ai/config.json
- **Share Example:** https://opencode.ai/s/4XP1fce5
