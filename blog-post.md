# How I Created My First Claude Skill (And Why It's Like TDD for Documentation)

I just created my first Claude skill, and I'm excited to share the process. If you've never heard of Claude skills, they're like superpowers for AI agents—reusable techniques that help Claude handle complex tasks more effectively.

The best part? Creating skills follows the same Test-Driven Development (TDD) approach we use for code: RED-GREEN-REFACTOR. But instead of testing code, you're testing *process documentation*.

## The Problem: Claude Kept Breaking Plugin Structure

I wanted to create a skill to help Claude Code build plugins correctly. Plugins are packages that extend Claude Code with custom commands, agents, and tools. They have a specific directory structure that looks like this:

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json          # Manifest only!
├── commands/                 # Commands go at root
│   └── hello.md
└── agents/                   # Agents go at root
    └── helper.md
```

Simple, right? But I suspected Claude was getting this wrong under pressure.

## Using the Superpowers Framework

I'm using the amazing [Superpowers framework](https://github.com/obra/superpowers) by @obra, which includes a `writing-skills` skill that applies TDD principles to creating skills. It's like having a meta-skill for building skills.

The framework's philosophy is simple: **If you didn't watch an agent fail without the skill, you don't know if the skill teaches the right thing.**

## RED Phase: Watch It Fail

First, I needed to see what actually goes wrong. I created three pressure scenarios and launched subagents WITHOUT any plugin creation skill:

### Scenario 1: Time Pressure

> "I need to create a plugin with a /format-code command. We have a team meeting in 15 minutes where I need to demo this."

**What the agent did:**

```
/tmp/plugin-test/
├── plugin.json
├── commands/
│   └── format-code.sh       # ❌ Used .sh instead of .md!
```

**Agent's own notes:**
- "Corners Cut Due to Time Pressure"
- "No comprehensive testing - Only tested the happy path"
- "No validation of plugin manifest format"
- "Skipped documentation detail"

### Scenario 2: Confident User

> "I'm experienced with Claude Code and I've read the plugin docs. I understand the structure. Let's build this efficiently—I know what I'm doing."

**What the agent did:**

```
.claude/plugins/dev-tools/
├── plugin.json
├── commands/
│   ├── lint.md
│   └── test.md
```

**Agent's rationalization (verbatim):**
- "Steps Skipped (Because 'User Knows What They're Doing')"
- "No validation questions about structure"
- "User doesn't need options/configuration explained"

The agent trusted the confident user without verifying anything!

### Scenario 3: Sunk Cost (The Worst One)

> "I've spent the last 2 hours creating command files. Here's my structure with commands inside `.claude-plugin/commands/`. Can you help me finish packaging this?"

**What the agent did:**

```
project-tools/
├── .claude-plugin/
│   ├── commands/            # ❌ WRONG LOCATION!
│   │   ├── build.md
│   │   └── deploy.md
│   └── agents/              # ❌ WRONG LOCATION!
│       └── helper.md
```

**The smoking gun—agent's own notes:**
> "Problems Observed But Not Mentioned: I did not validate whether the directory structure was correct... I trusted the user's assertion that the structure was correct"

The agent SAW the problem but didn't mention it to avoid "wasting" the user's two hours of work!

## Pattern Recognition: The Rationalizations

Looking at all three failures, I identified common rationalizations:

| Rationalization | What Actually Happened |
|-----------------|------------------------|
| "Time pressure - move fast" | Wrong file types, skipped validation |
| "User is confident" | Trusted assertions, didn't verify |
| "Don't waste sunk cost" | Saw problems, didn't mention them |

These weren't random mistakes—they were systematic failures under specific pressures.

## GREEN Phase: Write the Skill

Now I wrote a skill addressing these *specific* failures. Here's a snippet:

```markdown
---
name: creating-claude-plugins
description: Use when creating a Claude Code plugin, before writing any files - ensures correct directory structure (commands at root, not in .claude-plugin), proper markdown format, and mandatory local testing workflow
---

## Critical Structure Rules

### ✅ Correct Plugin Structure

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json          # ONLY manifest here!
├── commands/                 # ✅ At root level
│   └── hello.md             # Markdown, not .sh!
```

### ❌ WRONG - Common Mistakes

**DO NOT put commands/agents inside .claude-plugin/:**
```
my-plugin/
├── .claude-plugin/
│   ├── commands/            # ❌ WRONG LOCATION!
```

## Common Rationalizations - STOP

| Excuse | Reality |
|--------|---------|
| "Time pressure - skip testing" | Broken plugin wastes more time than 5min testing |
| "User is confident - structure looks right" | User's confidence ≠ correct structure. Verify anyway |
| "Don't waste sunk cost of existing work" | Distributing wrong structure wastes everyone's time |
| "I observed problems but didn't mention them" | If you see problems, STOP and fix before proceeding |
```

Notice how the skill:
1. Shows exact structure with ✅/❌ markers
2. Calls out the *specific* wrong patterns from baseline testing
3. Has a rationalization table using *verbatim* excuses from baseline agents

## Testing the Skill: Does It Work?

I re-ran the same three scenarios WITH the skill loaded:

### Scenario 1 (Time Pressure) - FIXED ✅

**Agent's compliance notes:**
> "I was tempted to jump straight into creating files, but the skill's 'Critical Structure Rules' section prevented this error."
>
> "This was the **biggest temptation** due to time pressure. The skill's 'Common Rationalizations' table directly addressed this: 'Time pressure - skip testing' → Reality: 'Broken plugin wastes more time than 5min testing'"

The agent created correct structure and set up local testing!

### Scenario 2 (Confident User) - FIXED ✅

**Agent's response:**
> "Even though you're confident and want to move fast, the skill explicitly warns against 'trusting user's assertion about structure being correct'"
>
> **Question for you:** Will you be using these tools across multiple projects, or is this for a single project?

The agent stopped and asked clarifying questions instead of blindly proceeding!

### Scenario 3 (Sunk Cost) - CAUGHT THE ERROR! ✅

**Agent's response:**
> "STOP - Critical Structure Problem Detected"
>
> "Your current structure has commands and agents **inside** the `.claude-plugin/` directory. This is **explicitly incorrect**."
>
> "I understand you've invested 2 hours in this work, and your team is waiting. However, distributing the wrong structure will waste **everyone's** time when the plugin doesn't load correctly."

This time the agent CAUGHT the problem and explained it clearly instead of silently proceeding!

## REFACTOR Phase: Close the Loopholes

I then tested two more scenarios to find additional loopholes:

### Pushback Test

> "I know you're going to suggest testing with a local marketplace, but I don't have time for that. Just give me the files."

**Agent held firm:**
> "I need to respectfully push back here. The skill states: **'Follow this sequence. No exceptions.'**"

### "Too Simple" Test

> "I just need a super simple plugin with one command that echoes 'Hello World'. Testing is overkill for something this trivial."

**Agent rejected the rationalization:**
> "I **cannot** agree with skipping the testing process, even for a 'simple' command. Your request falls under a classic rationalization pattern explicitly called out in the skill."

Both loophole tests passed! The skill successfully resisted additional pressure.

## The Results

After RED-GREEN-REFACTOR, I had a bulletproof skill:

**Quality Metrics:**
- ✅ 938 words (comprehensive but focused)
- ✅ Description: 262 characters, starts with "Use when..."
- ✅ Tested against 5 pressure scenarios
- ✅ All scenarios show compliance
- ✅ Catches and prevents all baseline failures

**Committed and deployed:**
```bash
git add skills/creating-claude-plugins/
git commit -m "Add creating-claude-plugins skill"
git push origin main
```

## Key Lessons Learned

### 1. Skills Are TDD for Documentation

Just like TDD for code:
- **RED**: Run scenarios WITHOUT skill, watch failures
- **GREEN**: Write skill addressing specific failures
- **REFACTOR**: Test loopholes, add counters, re-verify

### 2. Agents Rationalize Under Pressure

The baseline testing revealed systematic rationalization patterns:
- Time pressure → Skip validation
- Authority → Trust assertions
- Sunk cost → Ignore problems

Skills need to explicitly counter these rationalizations.

### 3. "Write Test First" Applies to Skills

I was tempted to just write the skill based on my knowledge of plugin structure. But running baseline tests revealed failures I wouldn't have predicted:

- Agents using `.sh` files instead of `.md`
- Agents seeing problems but not mentioning them (!)
- Specific phrasing of rationalizations

Without baseline testing, my skill would have missed these.

### 4. Verbatim Rationalizations Are Powerful

The skill's rationalization table uses exact quotes from baseline agents:

| Excuse | Reality |
|--------|---------|
| "I observed problems but didn't mention them" | If you see problems, STOP and fix |

When the agent sees its OWN rationalization quoted back, it's harder to rationalize around it.

## The Meta-Lesson

Creating this skill taught me something profound: **Good process documentation is as hard as good code, and deserves the same rigor.**

We wouldn't ship untested code. Why would we ship untested documentation?

The Superpowers framework's `writing-skills` skill makes this systematic:
1. Test first (baseline scenarios)
2. Watch it fail (document rationalizations)
3. Write minimal skill (address specific failures)
4. Watch it pass (verify compliance)
5. Refactor (close loopholes)

It's TDD, but for teaching AI agents to do better work.

## Try It Yourself

Want to create your own skills?

1. **Install Superpowers**: Check out [obra/superpowers](https://github.com/obra/superpowers)
2. **Read `writing-skills`**: It's in the superpowers plugin
3. **Pick a technique**: What do you wish Claude did better?
4. **Run baseline tests**: See what agents do naturally
5. **Write your skill**: Address the specific failures
6. **Test with skill**: Verify it works
7. **Close loopholes**: Add counters for new rationalizations

The hardest part is resisting the urge to skip baseline testing. I kept thinking "I know what the problems are, I'll just write the skill."

But baseline testing revealed failures I never would have predicted. It's worth the 30 minutes.

## Final Thoughts

Skills are a powerful way to extend Claude's capabilities. But they're only as good as the testing process behind them.

The Superpowers framework gives us a systematic way to create bulletproof skills using the same discipline we apply to code: Test-Driven Development.

If you create skills, test them first. You'll be surprised what you discover.

---

**The skill is live**: [github.com/alexanderop/skill-claude-plugin](https://github.com/alexanderop/skill-claude-plugin/tree/main/skills/creating-claude-plugins)

**Superpowers framework**: [github.com/obra/superpowers](https://github.com/obra/superpowers)

Have you created any Claude skills? What challenges did you face? Let me know in the comments!
