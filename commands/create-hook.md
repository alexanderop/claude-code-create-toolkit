---
description: Create and configure Claude Code hooks with reference documentation
argument-hint: [hook-type] [matcher] [command]
---

# /create-hook

## Purpose
Create and configure Claude Code hooks with reference documentation and interactive guidance.

## Contract
**Inputs:**
- `$1` — HOOK_TYPE (optional: PreToolUse, PostToolUse, UserPromptSubmit, Notification, Stop, SubagentStop, PreCompact, SessionStart, SessionEnd)
- `$2` — MATCHER (optional: tool name pattern, e.g., "Bash", "Edit|Write", "*")
- `$3` — COMMAND (optional: shell command to execute)

**Outputs:** `STATUS=<OK|FAIL> HOOK_FILE=<path>`

## Instructions

1. **Detect common use cases (hybrid approach):**
   - Scan user's request for keywords: "eslint", "prettier", "format", "lint", "typescript", "test", "commit"
   - If detected, ask: "I can set up a production-ready [TOOL] hook. Would you like to use the template or create a custom hook?"
   - If template chosen: Generate external script file in `.claude/hooks/` + settings.json entry
   - If custom or no match: Fall back to current workflow (steps 2-3)

2. **Determine hook configuration mode:**
   - If no arguments provided: Show interactive menu of hook types with examples
   - If HOOK_TYPE provided: Guide user through creating that specific hook
   - If all arguments provided: Create hook directly

3. **Validate inputs:**
   - HOOK_TYPE must be one of the valid hook events
   - MATCHER should be a valid tool name or pattern
   - COMMAND should be a valid shell command or path to external script
   - For external scripts: Ensure `.claude/hooks/` directory exists

4. **Reference documentation:**
   Review the Claude Code hooks documentation for best practices and examples:

   **Hook Events:**
   - `PreToolUse`: Runs before tool calls (can block them)
   - `PostToolUse`: Runs after tool calls complete
   - `UserPromptSubmit`: Runs when user submits a prompt
   - `Notification`: Runs when Claude Code sends notifications
   - `Stop`: Runs when Claude Code finishes responding
   - `SubagentStop`: Runs when subagent tasks complete
   - `PreCompact`: Runs before compact operations
   - `SessionStart`: Runs when session starts/resumes
   - `SessionEnd`: Runs when session ends

5. **Production-Ready Script Templates:**

   When user requests common integrations, generate these external scripts in `.claude/hooks/`:

   **ESLint Auto-Fix (`run-eslint.sh`):**
   ```bash
   #!/usr/bin/env bash
   # Auto-fix JavaScript/TypeScript files with ESLint after edits
   set -euo pipefail

   # Extract file path from Claude's JSON payload
   file_path="$(jq -r '.tool_input.file_path // ""')"

   # Only process JS/TS files
   [[ "$file_path" =~ \.(js|jsx|ts|tsx)$ ]] || exit 0

   # Auto-detect package manager (prefer project's lock file)
   if command -v pnpm >/dev/null 2>&1 && [ -f "pnpm-lock.yaml" ]; then
     PM="pnpm exec"
   elif command -v yarn >/dev/null 2>&1 && [ -f "yarn.lock" ]; then
     PM="yarn"
   else
     PM="npx"
   fi

   # Run ESLint with auto-fix from project root
   cd "$CLAUDE_PROJECT_DIR" && $PM eslint --fix "$file_path"
   ```

   **Prettier Format (`run-prettier.sh`):**
   ```bash
   #!/usr/bin/env bash
   # Format files with Prettier after edits
   set -euo pipefail

   file_path="$(jq -r '.tool_input.file_path // ""')"

   # Skip non-formattable files
   [[ "$file_path" =~ \.(js|jsx|ts|tsx|json|css|scss|md|html|yml|yaml)$ ]] || exit 0

   # Auto-detect package manager
   if command -v pnpm >/dev/null 2>&1 && [ -f "pnpm-lock.yaml" ]; then
     PM="pnpm exec"
   elif command -v yarn >/dev/null 2>&1 && [ -f "yarn.lock" ]; then
     PM="yarn"
   else
     PM="npx"
   fi

   cd "$CLAUDE_PROJECT_DIR" && $PM prettier --write "$file_path"
   ```

   **TypeScript Type Check (`run-typescript.sh`):**
   ```bash
   #!/usr/bin/env bash
   # Run TypeScript type checker on TS/TSX file edits
   set -euo pipefail

   file_path="$(jq -r '.tool_input.file_path // ""')"

   # Only process TypeScript files
   [[ "$file_path" =~ \.(ts|tsx)$ ]] || exit 0

   # Auto-detect package manager
   if command -v pnpm >/dev/null 2>&1 && [ -f "pnpm-lock.yaml" ]; then
     PM="pnpm exec"
   elif command -v yarn >/dev/null 2>&1 && [ -f "yarn.lock" ]; then
     PM="yarn"
   else
     PM="npx"
   fi

   # Run tsc --noEmit to check types without emitting files
   cd "$CLAUDE_PROJECT_DIR" && $PM tsc --noEmit --pretty
   ```

   **Run Affected Tests (`run-tests.sh`):**
   ```bash
   #!/usr/bin/env bash
   # Run tests for modified files
   set -euo pipefail

   file_path="$(jq -r '.tool_input.file_path // ""')"

   # Only run tests for source files (not test files themselves)
   [[ "$file_path" =~ \.(test|spec)\.(js|ts|jsx|tsx)$ ]] && exit 0
   [[ "$file_path" =~ \.(js|jsx|ts|tsx)$ ]] || exit 0

   # Auto-detect test runner and package manager
   if [ -f "vitest.config.ts" ] || [ -f "vitest.config.js" ]; then
     TEST_CMD="vitest related --run"
   elif [ -f "jest.config.js" ] || [ -f "jest.config.ts" ]; then
     TEST_CMD="jest --findRelatedTests"
   else
     exit 0  # No test runner configured
   fi

   if command -v pnpm >/dev/null 2>&1 && [ -f "pnpm-lock.yaml" ]; then
     PM="pnpm exec"
   elif command -v yarn >/dev/null 2>&1 && [ -f "yarn.lock" ]; then
     PM="yarn"
   else
     PM="npx"
   fi

   cd "$CLAUDE_PROJECT_DIR" && $PM $TEST_CMD "$file_path"
   ```

   **Commit Message Validation (`validate-commit.sh`):**
   ```bash
   #!/usr/bin/env bash
   # Validate commit messages follow conventional commits format
   set -euo pipefail

   # Extract commit message from tool input
   commit_msg="$(jq -r '.tool_input // ""')"

   # Check for conventional commit format: type(scope): message
   if ! echo "$commit_msg" | grep -qE '^(feat|fix|docs|style|refactor|perf|test|chore|ci|build|revert)(\(.+\))?: .+'; then
     echo "ERROR: Commit message must follow conventional commits format"
     echo "Expected: type(scope): description"
     echo "Got: $commit_msg"
     exit 2  # Block the commit
   fi
   ```

   **Corresponding settings.json entries:**
   ```json
   {
     "hooks": {
       "PostToolUse": [
         {
           "matcher": "Edit|Write",
           "hooks": [
             {
               "type": "command",
               "comment": "Auto-fix with ESLint",
               "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/run-eslint.sh"
             }
           ]
         }
       ]
     }
   }
   ```

6. **Best Practices for Hook Development:**

   **Always use shell safety headers:**
   ```bash
   #!/usr/bin/env bash
   set -euo pipefail  # Exit on error, undefined vars, pipe failures
   ```

   **Extract data from Claude's JSON payload:**
   ```bash
   # File path (Edit/Write tools)
   file_path="$(jq -r '.tool_input.file_path // ""')"

   # Command (Bash tool)
   command="$(jq -r '.tool_input.command // ""')"

   # Tool name
   tool_name="$(jq -r '.tool_name // ""')"

   # Full tool input
   tool_input="$(jq -r '.tool_input' | jq -c .)"
   ```

   **Use environment variables provided by Claude:**
   ```bash
   $CLAUDE_PROJECT_DIR   # Project root directory
   $CLAUDE_USER_DIR      # User's ~/.claude directory
   $CLAUDE_SESSION_ID    # Current session identifier
   ```

   **Efficient file extension filtering:**
   ```bash
   # Good: Use bash regex matching
   [[ "$file_path" =~ \.(ts|tsx)$ ]] || exit 0

   # Avoid: Spawning grep subprocess
   echo "$file_path" | grep -q '\.ts$' || exit 0
   ```

   **Package manager auto-detection pattern:**
   ```bash
   # Check lock files to match project's package manager
   if command -v pnpm >/dev/null 2>&1 && [ -f "pnpm-lock.yaml" ]; then
     PM="pnpm exec"
   elif command -v yarn >/dev/null 2>&1 && [ -f "yarn.lock" ]; then
     PM="yarn"
   else
     PM="npx"  # Fallback to npx
   fi
   ```

   **Exit codes matter:**
   ```bash
   exit 0  # Success: Allow operation to continue
   exit 1  # Error: Log error but don't block
   exit 2  # Block: Prevent operation in PreToolUse hooks
   ```

   **Performance considerations:**
   - Avoid heavy operations in tight loops (e.g., don't run full test suite on every file edit)
   - Use file extension checks to skip irrelevant files early
   - Consider async/background execution for slow operations
   - Cache results when possible (e.g., dependency checks)

   **When to use external scripts vs inline commands:**
   - **External scripts** (`.claude/hooks/*.sh`): Complex logic, multiple steps, reusable patterns
   - **Inline commands**: Simple one-liners, quick jq filters, logging

   Example inline command:
   ```json
   "command": "jq -r '.tool_input.command' >> ~/.claude/command-log.txt"
   ```

7. **Common use cases and examples (updated with best practices):**

   **Logging Bash commands:**
   ```json
   {
     "hooks": {
       "PreToolUse": [{
         "matcher": "Bash",
         "hooks": [{
           "type": "command",
           "comment": "Log all Bash commands with descriptions",
           "command": "set -euo pipefail; cmd=$(jq -r '.tool_input.command // \"\"'); desc=$(jq -r '.tool_input.description // \"No description\"'); echo \"[$(date -u +%Y-%m-%dT%H:%M:%SZ)] $cmd - $desc\" >> \"$CLAUDE_USER_DIR/bash-command-log.txt\""
         }]
       }]
     }
   }
   ```

   **Auto-format TypeScript files with Prettier:**
   ```json
   {
     "hooks": {
       "PostToolUse": [{
         "matcher": "Edit|Write",
         "hooks": [{
           "type": "command",
           "comment": "Auto-format TypeScript files after edits",
           "command": "set -euo pipefail; file_path=$(jq -r '.tool_input.file_path // \"\"'); [[ \"$file_path\" =~ \\.(ts|tsx)$ ]] || exit 0; cd \"$CLAUDE_PROJECT_DIR\" && npx prettier --write \"$file_path\""
         }]
       }]
     }
   }
   ```

   **Block sensitive file edits:**
   ```json
   {
     "hooks": {
       "PreToolUse": [{
         "matcher": "Edit|Write",
         "hooks": [{
           "type": "command",
           "comment": "Prevent edits to sensitive files",
           "command": "set -euo pipefail; file_path=$(jq -r '.tool_input.file_path // \"\"'); if [[ \"$file_path\" =~ \\.(env|secrets|credentials) ]] || [[ \"$file_path\" == *\"package-lock.json\" ]] || [[ \"$file_path\" == *.git/* ]]; then echo \"ERROR: Cannot edit sensitive file: $file_path\" >&2; exit 2; fi"
         }]
       }]
     }
   }
   ```

   **Desktop notifications:**
   ```json
   {
     "hooks": {
       "Notification": [{
         "matcher": "*",
         "hooks": [{
           "type": "command",
           "comment": "Send desktop notifications",
           "command": "if command -v notify-send >/dev/null 2>&1; then notify-send 'Claude Code' 'Awaiting your input'; elif command -v osascript >/dev/null 2>&1; then osascript -e 'display notification \"Awaiting your input\" with title \"Claude Code\"'; fi"
         }]
       }]
     }
   }
   ```

8. **Generic Patterns Reference:**

   **Pattern: Extract file path and check multiple extensions**
   ```bash
   set -euo pipefail
   file_path=$(jq -r '.tool_input.file_path // ""')
   [[ "$file_path" =~ \.(js|ts|jsx|tsx|py|go|rs)$ ]] || exit 0
   # Your command here
   ```

   **Pattern: Process multiple files from array**
   ```bash
   set -euo pipefail
   jq -r '.tool_input.files[]?' | while IFS= read -r file; do
     [[ -f "$file" ]] && echo "Processing: $file"
     # Your processing logic here
   done
   ```

   **Pattern: Conditional execution based on directory**
   ```bash
   set -euo pipefail
   file_path=$(jq -r '.tool_input.file_path // ""')
   # Only process files in src/ directory
   [[ "$file_path" =~ ^src/ ]] || exit 0
   # Your command here
   ```

   **Pattern: Extract and validate Bash command**
   ```bash
   set -euo pipefail
   cmd=$(jq -r '.tool_input.command // ""')
   # Block dangerous commands
   if [[ "$cmd" =~ (rm -rf|mkfs|dd|:(){:|:&};:) ]]; then
     echo "ERROR: Dangerous command blocked" >&2
     exit 2
   fi
   ```

   **Pattern: Background/async execution**
   ```bash
   set -euo pipefail
   file_path=$(jq -r '.tool_input.file_path // ""')
   # Run slow operation in background, don't block Claude
   (cd "$CLAUDE_PROJECT_DIR" && npm run build "$file_path" &> /tmp/build.log) &
   ```

   **Pattern: Conditional tool execution**
   ```bash
   set -euo pipefail
   tool_name=$(jq -r '.tool_name // ""')
   case "$tool_name" in
     Edit|Write)
       file_path=$(jq -r '.tool_input.file_path // ""')
       echo "File modified: $file_path"
       ;;
     Bash)
       cmd=$(jq -r '.tool_input.command // ""')
       echo "Command executed: $cmd"
       ;;
   esac
   ```

   **Pattern: Multi-tool chain with error handling**
   ```bash
   set -euo pipefail
   file_path=$(jq -r '.tool_input.file_path // ""')
   [[ "$file_path" =~ \.(ts|tsx)$ ]] || exit 0

   cd "$CLAUDE_PROJECT_DIR"

   # Run linter
   if ! npx eslint --fix "$file_path" 2>/dev/null; then
     echo "Warning: ESLint failed" >&2
   fi

   # Run formatter (always runs even if linter fails)
   npx prettier --write "$file_path" 2>/dev/null || true
   ```

   **Pattern: Cache validation results**
   ```bash
   set -euo pipefail
   file_path=$(jq -r '.tool_input.file_path // ""')
   cache_file="/tmp/claude-hook-cache-$(echo \"$file_path\" | md5sum | cut -d' ' -f1)"

   # Check cache freshness
   if [[ -f "$cache_file" ]] && [[ "$cache_file" -nt "$file_path" ]]; then
     cat "$cache_file"
     exit 0
   fi

   # Run expensive validation
   result=$(npx tsc --noEmit "$file_path" 2>&1)
   echo "$result" > "$cache_file"
   echo "$result"
   ```

   **Pattern: Cross-platform compatibility**
   ```bash
   set -euo pipefail

   # Detect OS and use appropriate commands
   case "$(uname -s)" in
     Darwin*)
       # macOS
       osascript -e 'display notification "Build complete"'
       ;;
     Linux*)
       # Linux
       notify-send "Build complete"
       ;;
     MINGW*|MSYS*|CYGWIN*)
       # Windows
       powershell -Command "New-BurntToastNotification -Text 'Build complete'"
       ;;
   esac
   ```

9. **Security considerations:**
   - Hooks run automatically with your environment's credentials
   - Always review hook implementation before registering
   - Be cautious with hooks that execute external commands
   - Avoid hardcoding sensitive data in hook commands
   - Use exit code 2 to block operations in PreToolUse hooks

10. **Hook creation workflow:**

   **For common tools (ESLint, Prettier, TypeScript, Tests):**
   1. Detect tool name from user request keywords
   2. Ask user: "I can set up a production-ready [TOOL] hook. Use template or create custom?"
   3. If template:
      - Create `.claude/hooks/` directory if needed
      - Generate appropriate script file (e.g., `run-eslint.sh`)
      - Make script executable (`chmod +x`)
      - Add settings.json entry with `$CLAUDE_PROJECT_DIR` reference
      - Verify package manager and dependencies exist
   4. Provide setup summary and next steps

   **For custom hooks:**
   1. Determine appropriate hook event for the use case
   2. Define matcher pattern (specific tool name, regex like "Edit|Write", or "*" for all)
   3. Write shell command that processes JSON input via stdin
   4. Always start with `set -euo pipefail` for safety
   5. Use `$CLAUDE_PROJECT_DIR` and other environment variables
   6. Test hook command independently before registering
   7. Choose storage location:
      - `.claude/settings.json` - Project-specific, committed to git
      - `.claude/settings.local.json` - Project-specific, not committed
      - `~/.claude/settings.json` - User-wide, all projects
   8. Register hook and reload with `/hooks` command

11. **Debugging hooks:**
   - Run `claude --debug` to see hook execution logs
   - Use `/hooks` command to verify hook registration
   - Check hook configuration in settings files
   - Test hook command standalone with sample JSON:
     ```bash
     echo '{"tool_name":"Edit","tool_input":{"file_path":"test.ts"}}' | .claude/hooks/run-eslint.sh
     ```
   - Review Claude Code output for hook execution errors
   - Use `jq` to inspect JSON data passed to hooks
   - Check exit codes (0=success/continue, 1=error/log, 2=block operation)
   - Verify file permissions (`chmod +x` for external scripts)
   - Check that `$CLAUDE_PROJECT_DIR` resolves correctly

12. **Output:**

   **Template mode:**
   - Create external script file in `.claude/hooks/[script-name].sh`
   - Generate settings.json configuration snippet
   - Make script executable
   - Print: `STATUS=OK HOOK_FILE=.claude/hooks/[script-name].sh SETTINGS=.claude/settings.json`
   - Show: Next steps (reload hooks, test the hook)

   **Interactive/Direct mode:**
   - Guide user through hook creation with prompts
   - Show the complete JSON configuration to add
   - Provide instructions for registering the hook
   - Print: `STATUS=OK HOOK_FILE=~/.claude/settings.json` or `STATUS=OK HOOK_FILE=.claude/settings.local.json`
   - Remind user to run `/hooks` to reload configuration

## Constraints
- External scripts must be executable (`chmod +x`)
- Hooks must be valid shell commands or script paths
- JSON structure must follow hooks schema
- PreToolUse hooks can block operations with exit code 2
- Hooks should be idempotent and handle errors gracefully
- Consider performance impact of hooks in tight loops
- Always use `set -euo pipefail` in bash scripts for safety
- Use `$CLAUDE_PROJECT_DIR` for project-relative paths
- Test hooks standalone before deploying

## Examples

**Template mode (recommended for common tools):**
```bash
# User: "Set up ESLint to run on file edits"
/create-hook
# Detects "ESLint", offers template
# → Creates .claude/hooks/run-eslint.sh
# → Adds PostToolUse hook to settings.json
# → Makes script executable
# → STATUS=OK HOOK_FILE=.claude/hooks/run-eslint.sh
```

**Interactive mode:**
```bash
/create-hook
# Shows menu of:
#  1. Common tools (ESLint, Prettier, TypeScript, Tests)
#  2. Hook types (PreToolUse, PostToolUse, etc.)
#  3. Custom hook creation
```

**Guided mode:**
```bash
/create-hook PostToolUse
# Guides through creating a PostToolUse hook
# Asks for: matcher, command/script, storage location
```

**Direct mode:**
```bash
/create-hook PreToolUse "Bash" "set -euo pipefail; cmd=\$(jq -r '.tool_input.command'); echo \"[\$(date -Iseconds)] \$cmd\" >> \"\$CLAUDE_USER_DIR/bash.log\""
# Creates hook configuration directly with best practices
```

## Reference
For complete documentation, see: https://docs.claude.com/en/hooks
