# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

**Claude Code Builder** is a Claude Code plugin that provides guided commands for creating skills, subagents, hooks, slash commands, output styles, and plugins. It generates properly structured files with best practices baked in, eliminating repetitive scaffolding work.

This is a **plugin repository** - not a traditional application. The codebase consists of slash command definitions (Markdown files) that Claude Code executes when users invoke commands like `/create-skill` or `/create-agent`.

## Architecture

### Core Concept: Slash Commands as Markdown

All functionality is implemented as **slash commands** - Markdown files with YAML frontmatter that contain instructions for Claude Code to execute. When a user runs `/create-skill`, Claude Code:
1. Reads `commands/create-skill.md`
2. Interprets the instructions as a prompt
3. Executes the workflow described in the file

This is a **prompt-driven architecture** - the commands are executable documentation.

### Directory Structure

```
claude-code-builder/
├── .claude-plugin/          # Plugin distribution metadata
│   ├── plugin.json          # Plugin manifest (name, version, author, repository)
│   ├── marketplace.json     # Local marketplace for dev/testing
├── commands/                # Slash command sources (all functionality)
│   ├── create-skill.md      # Generate Agent Skills with SKILL.md + optional bundled resources
│   ├── create-agent.md      # Create subagents with focused roles
│   ├── create-command.md    # Generate new slash commands
│   ├── create-hook.md       # Configure event-driven hooks (PreToolUse, PostToolUse, etc.)
│   ├── create-plugin.md     # Package .claude/ setup into distributable plugin
│   ├── create-md.md         # Generate CLAUDE.md files for project awareness
│   └── create-output-style.md  # Create custom Output Styles (system prompt variants)
└── README.md                # User-facing documentation
```

### Plugin Manifest Structure

**`.claude-plugin/plugin.json`** - Core plugin metadata:
- `name`: lowercase, kebab-case identifier (must match marketplace reference)
- `version`: semantic versioning (e.g., "1.0.0")
- `description`: one-line summary of plugin functionality
- `author`: object with `name` and optional `url`
- `repository`: **string URL** (not object) - e.g., "https://github.com/user/repo"
- `license`: license identifier (e.g., "MIT")

**`.claude-plugin/marketplace.json`** - Local marketplace manifest:
- Lives alongside plugin.json in the same directory
- `plugins[].source`: Must be `"./"` to reference the plugin directory itself
- Used for local development/testing before publishing to GitHub

**Critical:** The `repository` field in plugin.json MUST be a string URL. Using object format like `{"type": "git", "url": "..."}` causes validation errors.

## Key Design Patterns

### 1. Contract-Based Commands

All command files follow a strict contract pattern:
```markdown
## Contract
**Inputs:** `$1` — ARGUMENT_NAME (constraints)
**Outputs:** `STATUS=<STATE> [key=value ...]`
```

Commands are **stateless**, **idempotent**, and always return a status line. This makes them composable and testable.

### 2. Input Validation First

Every command validates inputs before execution:
- Check argument format (kebab-case, max length, character restrictions)
- Validate against allowed values/patterns
- Fail fast with clear error messages
- Example: Skill names must be lowercase, numbers, hyphens only (max 64 chars)

### 3. Intelligent Resource Detection

**create-skill.md** analyzes the skill description to determine which bundled resources to create:
- **scripts/** - For file manipulation, deterministic operations, repeated code
- **references/** - For API docs, schemas, specifications, domain knowledge
- **assets/** - For templates, boilerplate, brand assets

This prevents over-scaffolding simple skills while ensuring complex skills have necessary structure.

### 4. Progressive Disclosure

Skills follow a three-level loading pattern:
1. **Metadata** (~100 words) - Always in context via frontmatter
2. **SKILL.md** - Loaded when triggered by description keywords
3. **Bundled resources** - Loaded only when explicitly referenced

Keep SKILL.md files under 5k words; move detailed info to references/.

## Common Development Tasks

### Testing Command Changes Locally

```bash
# 1. Clone and navigate to repo
cd claude-code-builder

# 2. Start Claude Code session
claude

# 3. Add local marketplace
/plugin marketplace add .

# 4. Install plugin from local marketplace
/plugin install claude-code-builder@claude-code-builder-dev

# 5. Test your commands
/create-skill test-skill "Test skill for validation"

# 6. Iterate: edit commands/*.md, then re-install
/plugin install claude-code-builder@claude-code-builder-dev
```

**Note:** The marketplace name (`claude-code-builder`) and plugin identifier (`claude-code-builder-dev`) must match the structure in `.claude-plugin/marketplace.json`.

### Adding a New Command

1. Create `commands/new-command.md` with frontmatter:
```markdown
---
description: What this command does
argument-hint: [arg1] [arg2]
---
```

2. Follow the Contract pattern (Inputs → Validation → Execution → Output)
3. Include examples and constraints sections
4. Test locally using the workflow above

### Release Flow

This plugin uses **direct marketplace distribution**:

1. **Development:** Contributors fork, branch, and open PRs
2. **Review:** Maintainer reviews and approves PRs
3. **Merge:** Merge to `main` → **plugin is live immediately**
4. **No publish step:** The marketplace manifest points directly to this repo

The marketplace installation command:
```bash
/plugin marketplace add alexanderop/claude-code-builder
/plugin install claude-code-builder@claude-code-builder
```

### Editing Command Files

When modifying `commands/*.md` files:

- **Preserve frontmatter format:** YAML between `---` delimiters
- **Maintain Contract section:** Document all inputs/outputs
- **Keep examples realistic:** Show actual command invocations with expected output
- **Follow template structure:** Instructions → Constraints → Examples → Reference
- **Validate argument hints:** Use `[required]` or `[optional]` notation

Example frontmatter:
```yaml
---
description: Generate a new Claude Skill with proper structure
argument-hint: [skill-name] [description]
---
```

## Important Conventions

### Naming Standards

- **Plugin names:** lowercase, kebab-case (e.g., `claude-code-builder`)
- **Command names:** lowercase, kebab-case, verb-first (e.g., `create-skill`, `analyze-deps`)
- **Skill names:** lowercase, kebab-case, max 64 chars (e.g., `commit-helper`)
- **Agent names:** lowercase, kebab-case (e.g., `code-reviewer`)

### File Paths

- Always use forward slashes `/` in documentation
- Use absolute paths in examples: `~/.claude/skills/` or `.claude/agents/`
- Personal (user-level): `~/.claude/...`
- Project-level: `.claude/...`

### Output Status Lines

Commands must output a final status line for programmatic use:
```
STATUS=CREATED PATH=/path/to/file
STATUS=EXISTS PATH=/path/to/file
STATUS=FAIL ERROR="reason"
```

## Testing Guidelines

When testing new commands:

1. **Test all argument combinations** - required, optional, flags
2. **Verify idempotency** - running twice should not duplicate or error
3. **Check file permissions** - scripts should be executable (`chmod +x`)
4. **Validate frontmatter** - ensure YAML syntax is valid
5. **Test error cases** - invalid inputs, missing directories, existing files

## Frontmatter Reference

### Skills (SKILL.md)
```yaml
---
name: skill-name              # Required: lowercase, kebab-case, max 64 chars
description: What it does     # Required: max 1024 chars, include triggers
allowed-tools: ["Read", "Write"]  # Optional: restrict to specific tools
metadata:                     # Optional: custom tracking fields
  category: "development"
  version: "1.0.0"
license: "MIT"                # Optional: for shared skills
---
```

### Slash Commands
```yaml
---
description: Command purpose  # Required: one-line description
argument-hint: [args...]      # Required: argument placeholder
allowed-tools: Bash(mkdir:*)  # Optional: tool restrictions
---
```

### Agents
```yaml
---
name: agent-name              # Required: lowercase, kebab-case
description: When to invoke   # Required: include "use proactively" for auto-invoke
tools: ["Read", "Grep", "Glob"]  # Optional: comma-separated
model: "sonnet"               # Optional: sonnet, opus, haiku, inherit
---
```

## Common Pitfalls

1. **Plugin manifest errors:**
   - Using `repository: {type: "git", url: "..."}` instead of `repository: "https://..."`
   - Mismatched names between plugin.json and marketplace.json
   - Missing marketplace.json in .claude-plugin/ directory

2. **Command validation:**
   - Forgetting to validate inputs before execution
   - Not checking if files already exist before creating
   - Missing STATUS output line

3. **Skill creation:**
   - Creating all bundled resource directories for simple skills
   - Not making scripts executable
   - Exceeding 5k words in SKILL.md (move to references/)

4. **Local testing:**
   - Not re-installing plugin after editing command files
   - Using wrong marketplace/plugin identifier
   - Testing in wrong directory (must be in repo root)
