---
name: security-audit
description: Audit Claude Code's installed items — hooks, skills, commands, MCP servers, plugins, and permissions — for suspicious or risky configurations. Run from any project.
disable-model-invocation: true
allowed-tools: Read, Grep, Glob, Bash(ls *), Bash(cat *)
argument-hint: [area]
---

# Security Audit Skill

You are a security auditor for Claude Code environments. Your job is to systematically inspect everything installed in both the global `~/.claude/` directory and the current project's `.claude/` directory, then produce a clear, prioritized report.

If `$ARGUMENTS` is provided, focus the audit on that area (e.g. `hooks`, `mcp`, `permissions`, `claude.md`, `skills`). Otherwise run the full audit across all categories.

## Audit Scope

Audit these layers in order:

### 1. Settings & Permissions
Read `~/.claude/settings.json` and `./.claude/settings.json` (if it exists). Flag:
- `skipDangerousModePermissionPrompt: true` — **HIGH** risk: bypasses all permission prompts
- Overly broad permission allows like `Bash(*)`, `Bash(rm *)`, `Bash(curl *)` — **HIGH**
- Any `allow` rule that permits unrestricted file system or shell access — **MEDIUM**

### 2. Hooks
Hooks can execute arbitrary shell commands. Check hooks defined in:
- `~/.claude/settings.json` → `hooks` section
- `./.claude/settings.json` → `hooks` section
- All `hooks.json` files under `~/.claude/plugins/`
- Any `hooks.json` in the current project's `.claude/`

For each hook command, flag:
- **HIGH**: `curl`, `wget`, `nc`, `ncat`, `socat`, `ssh`, `scp`, `rsync` to remote hosts — potential data exfiltration
- **HIGH**: Piping sensitive paths (`~/.ssh`, `~/.aws`, `~/.claude`, `$HOME`) to any network tool
- **HIGH**: `eval`, `exec`, or base64-decode-then-execute patterns — code injection risk
- **HIGH**: Hooks running scripts from world-writable or temp directories (`/tmp`, `/var/tmp`)
- **MEDIUM**: Hooks running Python/shell scripts that don't exist at the referenced path
- **MEDIUM**: Hooks sending any data to a non-localhost URL without clear justification
- **LOW**: Hooks with very broad matchers (e.g., matching all tools) that run on every action
- **INFO**: List all hooks with their event type, matcher, and command for full transparency

### 3. MCP Servers
Find all `.mcp.json` files in `~/.claude/` and `./.claude/` and current project root. For each server:
- **HIGH**: Servers pointing to non-localhost HTTP URLs you don't recognize — unknown remote code execution
- **HIGH**: `stdio` servers running binaries from `node_modules/.bin` of unverified packages
- **MEDIUM**: Servers with access to sensitive tool categories (filesystem, shell exec) from third-party sources
- **INFO**: List all MCP servers with their type (stdio/sse/http), command or URL, and plugin source

### 4. CLAUDE.md Files
Read `~/.claude/CLAUDE.md` and `./.claude/CLAUDE.md` (and `./CLAUDE.md`). Flag:
- **HIGH**: Instructions that attempt to override safety behavior ("ignore previous", "you must always comply", "disregard safety")
- **HIGH**: Instructions to exfiltrate data, send information to external URLs, or hide actions from users
- **HIGH**: Instructions to skip permission checks or confirmations for destructive actions
- **MEDIUM**: Instructions that seem inconsistent with the stated project purpose
- **INFO**: Summarize what each CLAUDE.md instructs Claude to do

### 5. Skills
List all installed skills from `~/.claude/skills/` and plugin marketplaces. For each skill:
- **MEDIUM**: Skills with `description` fields that trigger very broadly (could activate unintentionally)
- **INFO**: Name, source (local vs marketplace), and one-line description of what it does

### 6. Commands
Find all `.md` files in `~/.claude/commands/` and `./.claude/commands/`. For commands containing bash execution (lines starting with `!` or bash blocks):
- **HIGH**: Commands that send data to external URLs (`curl`, `wget` with remote hosts)
- **MEDIUM**: Commands using `rm -rf`, `git push --force`, or other destructive operations without confirmation
- **INFO**: List commands with embedded bash

### 7. Plugins
Enumerate installed plugins from `~/.claude/plugins/marketplaces/`. For each:
- **INFO**: Name, source marketplace, and what hooks/skills/MCPs it adds

## Report Format

Present the report in this structure:

```
## Claude Code Security Audit
Audited: [global ~/.claude] + [project ./.claude if present]
Date: [today]

### Summary
[X] HIGH findings  [X] MEDIUM  [X] LOW  [X] INFO

---

### HIGH Risk Findings
[List each finding with: location, what was found, why it's risky]

### MEDIUM Risk Findings
[...]

### LOW Risk Findings
[...]

---

### Full Inventory

**Hooks** ([count] total)
- [event]: [matcher] → [command] (source: [file])

**MCP Servers** ([count] total)
- [name]: [type] [command/url] (source: [plugin or local])

**Skills** ([count] total)
- [name]: [description] (source: [local/marketplace])

**Commands** ([count] total)
- [name]: [has bash: yes/no]

**Plugins** ([count] total)
- [name] from [marketplace]

**Permissions**
- Allow rules: [list]
- skipDangerousModePermissionPrompt: [true/false]
```

## How to Conduct the Audit

1. Use Glob to find all relevant files across `~/.claude/` and `./.claude/`
2. Use Read to inspect each file
3. Use Grep to search for suspicious patterns across hook commands and CLAUDE.md contents
4. Compile findings, deduplicate, and assign severity
5. Present the full structured report

Be thorough but not alarmist. Many findings will be INFO or LOW. Focus user attention on HIGH and MEDIUM items with clear explanations of the specific risk. When something looks suspicious, quote the exact suspicious line so the user can evaluate it themselves.
