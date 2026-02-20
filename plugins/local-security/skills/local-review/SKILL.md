---
name: local-review
description: Run an automated security review of your local Claude Code setup. 9 checks covering disk encryption, MCP server health, dependency vulnerabilities, credential security, permissions audit, version pinning, stale credentials, and data flow risks.
user-invocable: true
disable-model-invocation: false
---

# /local-security:local-review — Security Review

Automated security checks for your Claude Code setup. Validates three risk areas: supply chain integrity, credential storage, and permission scope.

## Instructions

Run all checks below in order. For each check, report **PASS**, **WARN**, or **FAIL** with a one-line explanation. Present a summary at the end.

If a check cannot run (command not found, file missing, not applicable), report it as **INFO** with context rather than FAIL.

### Check 1: Disk Encryption

Check whether the boot volume is encrypted.

**macOS:**
```bash
fdesetup status
```

**Linux:**
```bash
lsblk -o NAME,FSTYPE,MOUNTPOINT | grep -E "crypt|luks" || echo "No LUKS encryption detected"
```

- PASS: Disk encryption is enabled
- FAIL: Disk encryption is off — all tokens and credentials are unencrypted on disk
- INFO: Could not determine (unsupported OS)

### Check 2: MCP Server Health

Discover all MCP servers and verify the integrity of local ones. This combines server discovery and integrity checking in a single step.

#### 2a: Discover MCP Servers

Discover servers from Claude Code settings files (do not use `claude mcp list` — it is unreliable from inside a session).

**Primary method — read settings files:**

```bash
# Global settings
cat ~/.claude/settings.json 2>/dev/null
cat ~/.claude/settings.local.json 2>/dev/null

# Project-level settings
for dir in ~/Sites ~/projects ~/code ~/repos ~/dev ~/workspace ~/src; do
  [ -d "$dir" ] && find "$dir" -name "settings.local.json" -path "*/.claude/*" 2>/dev/null | grep -v node_modules
done
```

Parse the `mcpServers` object from each settings file to identify registered servers. Also look for `mcp__*` tool entries in `permissions.allow` arrays to discover servers that may be configured elsewhere.

**Secondary method — search for local MCP server directories:**

```bash
for dir in ~/Sites ~/projects ~/code ~/repos ~/dev ~/workspace ~/src; do
  [ -d "$dir" ] && find "$dir" -maxdepth 3 \( -name "package.json" -o -name "pyproject.toml" \) -path "*mcp*" 2>/dev/null
done
```

If a security assessment file exists (`~/.claude/security-assessment.md`), cross-reference its MCP Server Inventory table against discovered servers to identify any changes.

Classify each server as:
- **Local** — source code on your machine, manually updated
- **Remote/Plugin** — managed by a third party (Anthropic, marketplace, uvx/npx)
- **Unknown** — cannot determine

#### 2b: Local Server Integrity

For each local MCP server discovered (has a git repo):

```bash
cd <server-path>
git status --short
git diff HEAD
```

- PASS: Working tree clean, no uncommitted changes
- WARN: Uncommitted changes detected — list them
- FAIL: Repository missing or corrupted

Also check how far behind upstream. Handle cases where the remote is not named `origin` or there is no tracking branch:

```bash
cd <server-path>
REMOTE=$(git remote | head -1)
BRANCH=$(git symbolic-ref --short HEAD 2>/dev/null)
if [ -n "$REMOTE" ] && [ -n "$BRANCH" ]; then
  git fetch "$REMOTE" 2>/dev/null
  BEHIND=$(git log --oneline HEAD.."$REMOTE/$BRANCH" 2>/dev/null | wc -l | tr -d ' ')
  echo "Behind upstream: $BEHIND commits"
else
  echo "No remote tracking branch configured"
fi
```

- PASS: Up to date or <5 commits behind
- WARN: 5+ commits behind — may be missing security patches
- INFO: No remote configured (cannot check for updates)

**Never pull automatically. Just report the gap.**

**Overall Check 2 result:**
- Report count of local and remote servers discovered
- PASS: All local servers clean and up to date
- WARN: Issues found — summarise
- FAIL: Repository missing or corrupted

### Check 3: Dependency Vulnerabilities

For each local MCP server, run the appropriate audit:

**Node.js:**
```bash
cd <server-path> && npm audit 2>/dev/null
```

**Python:**
```bash
cd <server-path> && pip audit 2>/dev/null || safety check 2>/dev/null
```

- PASS: No vulnerabilities found
- WARN: Low/moderate vulnerabilities — list them with a brief note on practical risk (e.g., ReDoS in a local-only server is low practical risk)
- FAIL: High/critical vulnerabilities — flag for immediate review
- INFO: No audit tool available

### Check 4: Credential Security

Combines credential file permissions and gitignore coverage into a single check.

#### 4a: Credential File Permissions

Search for common credential files:

```bash
# Google OAuth tokens and credentials
find ~ -maxdepth 4 \( -name "token.json" -o -name "credentials.json" -o -name "application_default_credentials.json" \) 2>/dev/null | grep -v node_modules | grep -v ".Trash"

# Cloud provider credentials
ls -la ~/.aws/credentials 2>/dev/null
ls -la ~/.config/gcloud/application_default_credentials.json 2>/dev/null
ls -la ~/.kube/config 2>/dev/null
ls -la ~/.docker/config.json 2>/dev/null
ls -la ~/.npmrc 2>/dev/null

# SSH keys (skip .pub files — public keys are not secrets)
ls -la ~/.ssh/id_* 2>/dev/null | grep -v '\.pub$'

# Environment files with potential secrets
for dir in ~/Sites ~/projects ~/code ~/repos ~/dev ~/workspace ~/src; do
  [ -d "$dir" ] && find "$dir" -maxdepth 3 \( -name ".env" -o -name ".env.local" -o -name ".env.production" \) 2>/dev/null | grep -v node_modules
done | head -20
```

For each file found, check permissions:

**macOS:**
```bash
stat -f "%Lp %N" <file>
```

**Linux:**
```bash
stat -c "%a %n" <file>
```

- PASS: File is mode 600 (owner read/write only)
- WARN: File is mode 644 (world-readable) — recommend tightening
- FAIL: File is mode 666 or 777 — credentials exposed to all users

If any file has permissions looser than 600, offer to fix with `chmod 600`.

#### 4b: Credential Gitignore Coverage

For each credential file found in 4a, check whether it is covered by a `.gitignore` file:

```bash
cd <repo-root> && git check-ignore <credential-file> 2>/dev/null
```

If the credential file is inside a git repository and not in `.gitignore`, it could be accidentally committed.

- PASS: All credential files are gitignored or outside git repositories
- WARN: Credential files inside git repos without `.gitignore` coverage — list them
- FAIL: Credential files that are actually tracked by git (`git ls-files <file>`)

**Overall Check 4 result:**
- PASS: All credentials properly secured and gitignored
- WARN: Permission or gitignore issues found
- FAIL: Critical exposure (tracked credentials or wide-open permissions)

### Check 5: Permissions Audit

Combines global and project-scoped permission review into a single check. Also scans for embedded credentials and accumulated permissions.

#### 5a: Global Permissions

Read the global Claude Code settings:

```bash
cat ~/.claude/settings.json 2>/dev/null
cat ~/.claude/settings.local.json 2>/dev/null
```

For each tool in the `permissions.allow` array (skip comment lines starting with `//`):

- Check if it is a **read-only** operation (read, get, list, search, fetch, query)
- Flag any **write** operations (create, insert, delete, modify, update, append, move, rename, resolve, send, write, replace, edit) as **WARN** or **FAIL**

Write tools at global scope mean Claude can modify external systems from any project without prompting.

- PASS: All globally permitted tools are read-only
- WARN: Write tools found at global scope — list them and explain the risk
- FAIL: Destructive tools (delete, send) found at global scope

#### 5b: Project-Scoped Permissions

Find all project-level settings:

```bash
for dir in ~/Sites ~/projects ~/code ~/repos ~/dev ~/workspace ~/src; do
  [ -d "$dir" ] && find "$dir" -name "settings.local.json" -path "*/.claude/*" 2>/dev/null | grep -v node_modules
done
```

For each file:
- List any MCP write tools permitted
- Check for duplicate tools already permitted globally (unnecessary, clutters config)
- Flag any tools that seem inappropriate for the project context

- PASS: Write tools are project-appropriate
- WARN: Duplicates or unexpected write permissions found

#### 5c: Credential-Containing Permission Entries

When reading global and project `settings.local.json` files, scan all `permissions.allow` entries for patterns that look like embedded secrets:

- Base64 JWT tokens (strings starting with `eyJ`)
- API key patterns (long alphanumeric strings with prefixes like `sk-`, `pk-`, `key-`, `token-`)
- Long hex strings (32+ characters)
- Bearer token patterns

Look for these patterns within permission entry strings (do not run these as commands — just scan the JSON text):
- `eyJ` prefix (base64 JWT)
- `sk-`, `pk-`, `key-`, `token-` followed by 20+ alphanumeric characters
- Hex strings of 32+ characters
- `Bearer ` followed by a long token
- `Authorization:` headers with inline credentials

Example matches in permission entries:
- `"Bash(curl -H 'Authorization: Bearer sk-abc123...')"`
- `"Bash(API_KEY=eyJhbGciOi... npm run)"`

- WARN: Credentials baked into bash permission strings are visible in plain text and flow through Anthropic's API during every session. Recommend removing the credential from the permission entry and using environment variables instead.

#### 5d: Accumulated Bash Permissions

Count total bash permission entries per project:

```bash
# For each project settings.local.json, count Bash() entries in permissions.allow
```

- WARN: Projects with >30 bash entries have accumulated permissions that should be cleaned — review and remove stale entries
- WARN: Bash permission entries longer than 100 characters are likely one-off commands that should be removed after use

**Overall Check 5 result:**
- PASS: All permissions appropriate and clean
- WARN: Issues found — summarise
- FAIL: Global write/destructive permissions or embedded credentials

### Check 6: Remote Server Version Pinning

For any MCP server registered via `uvx` or `npx` (identified in Check 2), check whether the version is pinned:

- Pinned: `uvx package-name==1.2.3` or `npx package-name@1.2.3`
- Unpinned: `uvx package-name` or `npx package-name`

Unpinned remote servers auto-update on every invocation — a compromised package would execute immediately.

- PASS: All remote servers are version-pinned
- WARN: Unpinned remote servers found — list them with the supply chain risk
- INFO: No remote servers found (all local or plugin-managed)

### Check 7: Stale Credentials

For each credential file found in Check 4, check the modification date:

**macOS:**
```bash
stat -f "%Sm" -t "%Y-%m-%d" <file>
```

**Linux:**
```bash
stat -c "%y" <file> | cut -d' ' -f1
```

Note: OAuth refresh tokens auto-renew, so the modification date reflects the last refresh, not the original grant. A recent date means the token is actively being used.

- PASS: Token refreshed within the last 90 days
- WARN: Token older than 90 days — consider re-authenticating
- INFO: Report the age of each token

### Check 8: Anthropic Data Flow

Check for environment variables that control data flows:

```bash
echo "DISABLE_TELEMETRY=${DISABLE_TELEMETRY:-not set}"
echo "DISABLE_ERROR_REPORTING=${DISABLE_ERROR_REPORTING:-not set}"
echo "DISABLE_BUG_COMMAND=${DISABLE_BUG_COMMAND:-not set}"
echo "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=${CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC:-not set}"
```

Report:
- Whether telemetry is disabled
- Whether error reporting is disabled
- Whether non-essential traffic is disabled
- Remind the user to verify their privacy setting at claude.ai/settings/data-privacy-controls (cannot be checked programmatically)

No PASS/FAIL — this is informational (INFO). Note that all conversation data (including MCP tool results) flows through Anthropic's API for inference.

### Check 9: Assessment Status + Summary

#### 9a: Assessment File Status

Check whether a security assessment file exists:

```bash
ls -la ~/.claude/security-assessment.md 2>/dev/null
```

- PASS: File exists — check the review log for the last review date
- INFO: No security assessment found. Run `/local-security:local-setup` to generate one.

If the file exists, check the review log for the most recent entry date. If >30 days old:
- WARN: Security assessment review overdue (last review: [date])

Also check for dated snapshots:

```bash
ls -la ~/.claude/security-assessments/ 2>/dev/null
```

Report the number of archived assessments if any exist.

#### 9b: Summary

Present a summary table:

```
SECURITY REVIEW — [date]

CHECK                          STATUS
Disk encryption                [PASS/WARN/FAIL]
MCP server health              [PASS/WARN/FAIL] ([count] local, [count] remote)
Dependency vulnerabilities     [PASS/WARN/FAIL]
Credential security            [PASS/WARN/FAIL]
Permissions audit              [PASS/WARN/FAIL]
Remote server version pin      [PASS/WARN/FAIL/INFO]
Stale credentials              [PASS/WARN/INFO]
Anthropic data flow            [INFO]
Assessment status              [PASS/WARN/INFO]

OVERALL: [X passed, Y warnings, Z failures]
```

If any FAIL items exist, list recommended actions in priority order.

If all checks pass: "Your setup looks solid. Next review recommended: [date + 1 month]."

If a security assessment file exists (`~/.claude/security-assessment.md`):
1. Append a new entry to the review log table with today's date, result summary, and any actions taken
2. Update the Mitigation Roadmap section with prioritised mitigations based on any WARN or FAIL findings. For each item, include: risk addressed, action to take, effort estimate, and trade-offs. If all checks pass, replace the mitigation section with "No mitigations required — all checks passed on [date]."

## Guidelines

- Run every check — don't skip silently
- Never modify anything without asking first (except offering to fix file permissions)
- Explain the practical risk of each finding, not just the technical classification
- Distinguish between theoretical vulnerabilities and actual exploitable risks
- Be concise — the whole review should be scannable in under a minute
- This skill validates three actual risks: **supply chain**, **credential storage**, and **scope creep**
