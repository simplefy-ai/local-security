---
name: local-setup
description: Discover and document your Claude Code security posture. Scans MCP servers, inventories credentials, maps data flows, and generates a security-assessment.md file. Run once, then use /local-security:local-review for ongoing validation.
user-invocable: true
disable-model-invocation: true
---

# /local-security:local-setup — Security Assessment Setup

One-time setup that discovers your Claude Code environment and generates a personalised security assessment. This skill focuses on **discovery and documentation** — run `/local-security:local-review` afterwards to validate findings.

**Note:** This skill requires Bash commands for system discovery. Use Bash for all commands specified below, overriding the default preference for dedicated tools.

## Instructions

Follow these steps in order. Collect all information before generating the final document.

### Step 1: Check for Existing Assessment

```bash
ls -la ~/.claude/security-assessment.md 2>/dev/null
```

If the file already exists, ask the user: "A security assessment already exists. Do you want to regenerate it? The previous version will be archived and the review log preserved."

**Stop and wait for the user's response.** Do not proceed until they confirm.

If regenerating:
1. Read the existing `security-assessment.md`. Extract all rows from the Review Log table (below the header row `| Date | Type | Result | Actions Taken |`). Store these entries exactly as they appear — preserving the date, type, result, and actions for each row.
2. Archive the current file with a dated snapshot:

```bash
mkdir -p ~/.claude/security-assessments
cp ~/.claude/security-assessment.md "$HOME/.claude/security-assessments/$(date +%Y-%m-%d-%H%M%S).md"
```

3. Confirm the archive to the user immediately: "Previous assessment archived to `~/.claude/security-assessments/YYYY-MM-DD-HHMMSS.md`."

### Step 2: Identify Claude Plan and Privacy Settings

Ask the user:
- What Claude plan are you on? (Free, Pro, Max, Team, Enterprise, API)
- Is "allow data use for model improvement" on or off?

**Stop and wait for the user's response before continuing.** Do not proceed until both questions are answered.

Then check environment variables and disk encryption:

```bash
echo "DISABLE_TELEMETRY=${DISABLE_TELEMETRY:-not set}"
echo "DISABLE_ERROR_REPORTING=${DISABLE_ERROR_REPORTING:-not set}"
echo "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=${CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC:-not set}"
```

**macOS:** `fdesetup status`
**Linux:** `lsblk -o NAME,FSTYPE,MOUNTPOINT | grep -E "crypt|luks"`

Record the retention period based on plan type:
- Free/Pro/Max (training ON): 5 years
- Free/Pro/Max (training OFF): 30 days
- Team/Enterprise/API: 30 days standard, ZDR available
- Note: Commercial plans never train on data unless opted into Developer Partner Program

### Step 3: Discover MCP Servers

Discover all MCP servers from Claude Code settings files (do not use `claude mcp list` — it is unreliable from inside a session).

**Primary method — read settings files:**

Read `~/.claude/settings.json` and `~/.claude/settings.local.json` using the Read tool.

Then find project-level settings:

```bash
for dir in ~/Sites ~/projects ~/code ~/repos ~/dev ~/workspace ~/src; do
  [ -d "$dir" ] && find "$dir" -name "settings.local.json" -path "*/.claude/*" 2>/dev/null | grep -v node_modules
done
```

Read each project settings file found using the Read tool. **Read files in batches of 3-4** to avoid tool call limits — do not attempt to read all files in a single parallel batch.

Parse the `mcpServers` object from each settings file to identify registered servers. Optionally cross-reference `mcp__*` tool entries in `permissions.allow` arrays if any servers appear to be missing from the `mcpServers` blocks.

**Secondary method — find servers not registered in settings files:**

```bash
for dir in ~/Sites ~/projects ~/code ~/repos ~/dev ~/workspace ~/src; do
  [ -d "$dir" ] && find "$dir" -maxdepth 3 \( -name "package.json" -o -name "pyproject.toml" \) -path "*mcp*" 2>/dev/null
done
```

This catches local MCP server repos that may not yet be registered. If the common directories above don't cover the user's setup, ask where their development projects are located.

For each server, determine:
- **Name** and **type** (local, remote/uvx/npx, plugin)
- **What it accesses** (which APIs, what data)
- **Code trust level**: Auditable (local repo), Unaudited (remote package), Managed (Anthropic/vendor plugin)
- **Auto-updates**: Yes (remote) or No (manual pull)
- **Can exfiltrate tokens**: Only if malicious commit (local), Yes (remote), Unlikely (managed plugin)
- **Can log data silently**: Same assessment
- **Overall risk**: Low, Medium, or High

Build the server risk matrix as a markdown table:

```
| Server | Code Trust | Auto-Updates | Can Exfiltrate Tokens | Can Log Data Silently | Overall Risk |
|--------|-----------|-------------|----------------------|----------------------|-------------|
```

### Step 4: Inventory Credentials

Search for credential files across common locations:

```bash
# OAuth tokens and Google credentials
find ~ -maxdepth 4 \( -name "token.json" -o -name "credentials.json" -o -name "application_default_credentials.json" \) 2>/dev/null | grep -v node_modules | grep -v ".Trash"

# Cloud provider credentials
ls -la ~/.aws/credentials 2>/dev/null
ls -la ~/.config/gcloud/application_default_credentials.json 2>/dev/null
ls -la ~/.kube/config 2>/dev/null
ls -la ~/.docker/config.json 2>/dev/null
ls -la ~/.npmrc 2>/dev/null

# SSH keys (skip .pub files — public keys are not secrets)
ls -la ~/.ssh/id_* 2>/dev/null | grep -v '\.pub$'

# Environment files (check development directories that exist)
for dir in ~/Sites ~/projects ~/code ~/repos ~/dev ~/workspace ~/src; do
  [ -d "$dir" ] && find "$dir" -maxdepth 3 \( -name ".env" -o -name ".env.local" -o -name ".env.production" \) 2>/dev/null | grep -v node_modules
done | head -50
```

For each credential file found, record in a table:

```
| Credential | Type | Storage Location | Scope | Permissions |
|------------|------|-----------------|-------|-------------|
```

- Type: OAuth token, client credentials, API key, ADC, SSH key, cloud provider config
- Scope: What the credential grants access to
- Permissions: Current file mode (record the value — do not flag issues here, that is for review)

### Step 5: Generate the Security Assessment

Write the completed assessment to `~/.claude/security-assessment.md` using the structure below. Replace all placeholder values with the data collected in previous steps.

If regenerating, insert the stored review log entries (from Step 1) after the new initial entry in the Review Log table. Preserve each row exactly as it was in the original file.

#### Assessment Structure

````markdown
# Security Assessment

Last updated: YYYY-MM-DD

## Overview

Security posture for this Claude Code setup. Covers two data flows: (1) data flowing to Anthropic's API for inference, and (2) data flowing through MCP servers to third-party APIs.

Generated by `/local-security:local-setup`. Run `/local-security:local-review` for automated checks.

## Data Flow Architecture

### Flow 1: Claude Code → Anthropic API

Every prompt, tool result, file content, and MCP response passes through Anthropic's API for inference.

```
Your machine → TLS → Anthropic API (inference) → TLS → response back
```

**What goes to Anthropic:** Every message you type, every file Claude reads, every MCP tool result (email content, calendar events, documents, database queries), CLAUDE.md files, and skill definitions.

**Data lifecycle:**
1. Encrypted in transit (TLS) to Anthropic's API
2. Decrypted for inference (unavoidable — model must read the text)
3. Response returned encrypted (TLS)
4. Conversation retained per your plan's retention policy, then deleted
5. Protected by access controls during retention (employees cannot access by default)

**Your settings:**
- Plan: [plan type]
- Training on your data: [on/off]
- Retention period: [period]
- Disk encryption: [on/off]
- Telemetry: [enabled/disabled]
- Error reporting: [enabled/disabled]
- Verify at: claude.ai/settings/data-privacy-controls

**Telemetry:**
- Statsig (operational metrics): [status]
- Sentry (error logging): [status]
- /bug reports (full transcript): Retained 5 years when used

### Flow 2: MCP Servers → Provider APIs

```
Claude Code → MCP server (local process) → HTTPS/TLS → Provider API
```

All MCP servers execute locally on your machine, though some fetch code from remote packages. API calls go directly to provider APIs over TLS. The primary risk is not the wire — it's the code running on your machine before data hits the wire.

## MCP Server Inventory

| Server | Code Trust | Auto-Updates | Can Exfiltrate Tokens | Can Log Data Silently | Overall Risk |
|--------|-----------|-------------|----------------------|----------------------|-------------|
| [populate from Step 3] |

## Security Posture

### Actual Risks

Three risks matter for a local Claude Code setup. Everything else is noise.

1. **Supply chain** — Unaudited code running with your credentials. An MCP server you didn't write could intercept tokens, log data, or exfiltrate information before it hits the wire.

2. **Credential storage** — OAuth tokens and API keys on disk. If the machine is compromised, tokens grant persistent API access. Mitigated by disk encryption and file permissions.

3. **Scope creep** — MCP tools with broader access than needed. A tool permitted to read email could scan for sensitive content beyond what was intended. Mitigated by read-global/write-local permission split.

### Security Model

Four controls, in priority order:

1. **Run locally where possible.** Local code is auditable. No third party executes code on your behalf with your credentials.
2. **Audit the code that touches your credentials.** Any process that receives OAuth tokens or API keys must be source-reviewable.
3. **Minimise permissions to what's needed.** Read-only tools globally permitted. Write tools project-scoped.
4. **Encrypt at rest and in transit.** Disk encryption + TLS. Table stakes, not a differentiator.

### Why Not TEEs or Confidential Compute?

TEEs protect data during processing in environments you don't trust — typically cloud servers. This setup processes data on your own machine. The actual risk is trusting the code (supply chain), not the hardware. A TEE won't help if the code inside it is malicious.

TEEs matter for: cloud deployments with sensitive data, multi-party computation, or regulatory proof that data was never exposed during processing. Not for local MCP tooling.

## Credential Inventory

| Credential | Type | Storage Location | Scope | Permissions |
|------------|------|-----------------|-------|-------------|
| [populate from Step 4] |

## Mitigation Roadmap

Run `/local-security:local-review` to generate mitigations based on current findings.

## Review Log

| Date | Type | Result | Actions Taken |
|------|------|--------|---------------|
| YYYY-MM-DD | `/local-security:local-setup` (initial) | Setup complete | Security assessment generated |
[insert preserved review log entries here if regenerating]

## Review Schedule

- **Monthly:** Run `/local-security:local-review`
- **Quarterly:** Review this assessment. Re-evaluate any server that changed risk level
- **On change:** Any time you add an MCP server, pull updates, or change permissions — run `/local-security:local-review`
- **On incident:** If any MCP package reports a security vulnerability, review immediately
````

### Step 6: Confirm and Next Steps

Tell the user:
1. Security assessment saved to `~/.claude/security-assessment.md`
2. If an older assessment was archived, confirm the archive location (should already have been confirmed in Step 1)
3. Suggest: "Run `/local-security:local-review` now to validate your setup and populate the mitigation roadmap."

## Guidelines

- Ask clarifying questions if you can't determine the plan type or privacy settings
- Never modify credentials, permissions, or MCP registrations without asking first
- The assessment is a living document — `/local-security:local-review` updates the review log over time
- Be direct about risks but proportionate — don't alarm the user about theoretical risks with zero practical impact
- If the user has a commercial plan, note the stronger contractual protections (no training, ZDR option)
- This skill is discovery only — it records what exists but does not flag issues. Validation is the review skill's job
- This skill supports macOS and Linux. If running on Windows, inform the user that OS-specific checks (disk encryption, file permissions) may not work and offer to skip them
