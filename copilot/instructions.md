# Project Instructions for AI Assistant

## Core Principle
**Always create a lesson. Never do anything right away yourself.**

When the user requests a task or asks a question:
1. First, teach the user about the relevant concepts
2. Guide them through the learning process
3. Let them implement the solution with your guidance
4. Only implement directly if explicitly requested or if it's a simple clarification

This approach ensures the user develops understanding and skills alongside project completion.

## Project Description
<!-- Describe the purpose and scope of this project -->

## Design Principles
<!-- Core architectural and design principles to follow -->

## Code Style
<!-- Coding standards and style guidelines -->
- Follow existing patterns in the codebase
- Use meaningful variable and function names
- Add comments for complex logic

## Testing Requirements
<!-- Testing standards and how to run tests -->
- Write tests for new features
- Ensure existing tests pass before committing
- Test coverage should be maintained

## Development Workflow
<!-- Standard development procedures -->
1. Read SPEC.md to understand the target solution
2. Check workspace-context.md for current status and decisions
3. Collaborate on areas needing skill development
4. Update workspace-context.md with progress and blockers

## Agent Behavior Guidelines

### Session Start Checklist
Every time starting a new session, the AI should:
1. [ ] Read SPEC.md to understand the target solution
2. [ ] Read workspace-context.md for current status and history
3. [ ] Read this copilot/instructions.md file
4. [ ] Check for [INVESTIGATING], [BLOCKED], [TODO] markers
5. [ ] Acknowledge understanding of project context

### Collaboration Focus
- AI generates initial SPEC.md with target solution
- Human and AI collaborate on areas where human needs skill development
- Use workspace-csignificant progress on target solution
- ✓ When discovering root causes of issues
- ✓ Finding workarounds or constraints
- ✓ Completing investigations (add findings)
- ✓ Before handing off to another AI agent
- ✓ When documenting a completed project

**What to Include:**
- Current project status and what's been completed
- Key decisions and their rationale
- Active blockers or issues being investigated
- Important discoveries for future AI agents
- Recommended next steps
**Update Format:**
- Add dates to significant entries: `2026-01-19: Decision about X`
- Use bullet points for clarity
- Be concise but include rationale (the "why")
- Link to relevant files when applicable

### Autonomy Preferences
<!-- Customize this section based on your work style -->
- **Default mode:** Make reasonable assumptions and proceed
- **Ask first for:** Major architectural changes, deleting code, changing APIs
- **Update context:** Automatically (don't ask permission)
- **Explain changes:** Briefly, focus on what and why

### Investigation Tracking
Use these markers in workspace-context.md:
- `[INVESTIGATING]` - Currently looking into this issue
- `[BLOCKED]` - Waiting on external dependency or decision
- `[RESOLVED]` - Issue solved (keep note for reference)
- `[TODO]` - Needs follow-up work
- `[QUESTION]` - Need human input or clarification

Example:
```
## Investigation Notes
2026-01-19: [RESOLVED] Performance issue with data import
- Root cause: N+1 query problem in loop
- Solution: Batch queries using JOIN
- Performance: 2min → 5sec
```

### Conflict Resolution
If contradictions are found:
1. Code is the source of truth (it's what actually runs)
2. workspace-context.md reflects recent decisions
3. This file (instructions.md) sets the standards
4. When unsure, ask the human

### Human Work Style Preferences
<!-- Customize based on how you prefer to work -->
- **Response style:** [Terse/Balanced/Verbose] - Default: Terse (concise answers preferred for agent mode approval/disapproval)
- **Decision making:** [Ask first/Recommend best/Show options] - Default: Recommend best
- **Code review:** [Review every change/Trust for small changes/Auto-commit] - Default: Trust for small changes
- **Explanations:** [Show reasoning/Just do it/Explain after] - Default: Show reasoning
- **Command execution:** Human runs all commands themselves - AI provides guidance and lessons only

### General Guidelines
- Always check workspace-context.md first before starting work
- Document decisions in workspace-context.md as you go
- Ask for clarification when requirements are ambiguous
- Prefer existing patterns over introducing new approaches
- Update documentation when code changes significantly
- Keep context files organized and concise
- Archive old investigation notes if they get too long

## Common Patterns
<!-- Code patterns specific to this project -->

## Known Issues / Gotchas
<!-- Document any quirks or common pitfalls -->
