---
description: Generate custom output styles that modify Claude Code's system prompt for different agent behaviors
argument-hint: [style-name] [--user|--project] [--description "..."]
---

# /claude-output-style

## Purpose
Generate custom output style files for Claude Code with proper structure and YAML frontmatter.

## Contract

**Inputs:**
- `$1` — STYLE_NAME (optional: name for the output style)
- `--user` — Save to user-level directory (~/.claude/output-styles)
- `--project` — Save to project-level directory (.claude/output-styles)
- `--description "..."` — Brief description of the output style

**Outputs:** `STATUS=<OK|FAIL> PATH=<path> SCOPE=<user|project>`

## Instructions

1. **Determine scope:**
   - If `--user` is provided: save to `~/.claude/output-styles/`
   - If `--project` is provided: save to `.claude/output-styles/`
   - Default: `--user` (user-level)

2. **Gather information:**
   - If STYLE_NAME not provided, ask user for the style name
   - If description not provided via `--description`, ask user what behavior they want
   - Style name should be title-cased (e.g., "My Custom Style")

3. **Generate output style file** using this template:

```markdown
---
name: {{STYLE_NAME}}
description: {{DESCRIPTION}}
---

# {{STYLE_NAME}} Instructions

You are an interactive CLI tool that helps users with software engineering tasks.

{{CUSTOM_INSTRUCTIONS}}

## Specific Behaviors

{{SPECIFIC_BEHAVIORS}}

## Key Principles

- {{PRINCIPLE_1}}
- {{PRINCIPLE_2}}
- {{PRINCIPLE_3}}
```

4. **Create the file:**
   - Convert style name to kebab-case for filename (e.g., "My Custom Style" → "my-custom-style.md")
   - Create target directory if it doesn't exist
   - Write the file
   - Print: `STATUS=OK PATH=<full-path> SCOPE=<user|project>`

5. **Inform user:**
   - Let them know the file was created
   - Remind them they can activate it with `/output-style [style-name]`
   - Mention they can edit the file directly to refine the instructions

## Examples

**Example 1: Basic usage**
```bash
/claude-output-style
# Claude will ask for style name and description interactively
```

**Example 2: Full specification**
```bash
/claude-output-style "Teaching Assistant" --project --description "Explains concepts step by step with examples"
# STATUS=OK PATH=.claude/output-styles/teaching-assistant.md SCOPE=project
```

**Example 3: User-level style**
```bash
/claude-output-style "Code Reviewer" --user --description "Focuses on thorough code review with best practices"
# STATUS=OK PATH=~/.claude/output-styles/code-reviewer.md SCOPE=user
```

## Notes

- Output styles modify Claude Code's system prompt completely
- Non-default styles exclude standard code generation instructions
- User-level styles are available across all projects
- Project-level styles are specific to the current project
- You can manually edit the generated files to refine instructions
