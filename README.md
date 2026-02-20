# local-security

A [Claude Code plugin](https://docs.anthropic.com/en/docs/claude-code) that audits the security of your local Claude Code environment.

If you use Claude Code with MCP servers, OAuth tokens, API keys, or third-party plugins, this plugin helps you understand what's exposed and whether it's properly protected.

## Install

Run these commands inside Claude Code:

```
/plugin marketplace add simplefy-ai/local-security
/plugin install local-security@simplefy-ai
```

Then restart Claude Code.

## How it works

Two skills with different jobs:

| Skill | What it does | When to run |
|-------|-------------|-------------|
| `/local-security:local-setup` | Scans your environment and creates a security assessment document | Once, or when your setup changes significantly |
| `/local-security:local-review` | Runs automated checks and flags issues | Monthly, or after adding servers/credentials/permissions |

**Setup** discovers what you have — MCP servers, credential files, privacy settings, data flows — and writes everything to `~/.claude/security-assessment.md`. It documents but does not judge. Previous assessments are archived with timestamps.

**Review** validates what setup found. It checks:

1. Disk encryption (FileVault / LUKS)
2. MCP server health — are local servers clean, up to date?
3. Dependency vulnerabilities — `npm audit` / `pip audit`
4. Credential security — file permissions and gitignore coverage
5. Permissions audit — write tools at global scope, secrets embedded in permission entries, stale one-off bash commands piling up
6. Remote server version pinning — unpinned packages are a supply chain risk
7. Stale credentials — tokens that haven't refreshed in 90+ days
8. Anthropic data flow — telemetry, error reporting, privacy settings
9. Assessment status — is the security assessment file current?

Each check reports PASS, WARN, or FAIL. Review also populates a mitigation roadmap in the assessment file with prioritised actions for any issues found.

## What risks does this cover?

Three things that actually matter for a local Claude Code setup:

1. **Supply chain** — Is the code running with your credentials auditable and trusted? Unaudited MCP servers pulled from PyPI or npm could intercept tokens or log data.
2. **Credential storage** — Are your OAuth tokens, API keys, and SSH keys properly protected on disk? File permissions and gitignore coverage matter.
3. **Scope creep** — Do your MCP tools have broader access than they need? Write tools at global scope mean Claude can modify external systems from any project.

## Requirements

- Claude Code (CLI)
- macOS or Linux
- Git (for local MCP server integrity checks)
- npm or pip (for dependency auditing — optional)

## Licence

MIT — see [LICENSE](plugins/local-security/LICENSE).

Built by [simplefy.ai](https://simplefy.ai)
