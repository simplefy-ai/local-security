# local-security

Security review and hardening for Claude Code.

Two skills that help you understand and maintain the security of your local Claude Code setup — MCP servers, credentials, permissions, and data flows.

## Install

```bash
/plugin marketplace add simplefy-ai/local-security
/plugin install local-security@simplefy-ai
```

## Skills

### `/local-security:local-setup`

One-time discovery. Scans your environment and generates a personalised security assessment at `~/.claude/security-assessment.md`. Focuses on **discovering and documenting** your setup — it does not flag issues or validate findings.

What it does:
- Discovers all MCP servers and classifies their trust level
- Inventories credential files and records their current state
- Maps your Anthropic privacy settings and data flows
- Generates a risk matrix and credential inventory
- Archives previous assessments as dated snapshots (`~/.claude/security-assessments/YYYY-MM-DD.md`)

Run once. Then use `/local-security:local-review` to validate and generate mitigations.

### `/local-security:local-review`

Automated security validation. Run monthly or after any configuration change. This is where issues get flagged and mitigations are recommended.

| Check | What it validates |
|-------|-------------------|
| Disk encryption | FileVault (macOS) or LUKS (Linux) enabled |
| MCP server health | Discovery, git status, uncommitted changes, upstream drift |
| Dependency vulnerabilities | `npm audit` / `pip audit` on local MCP servers |
| Credential security | File permissions (mode 600) and gitignore coverage |
| Permissions audit | Global/project scope, secrets in permission entries, stale one-off bash commands |
| Remote server version pin | Unpinned `uvx`/`npx` servers are a supply chain risk |
| Stale credentials | Tokens older than 90 days flagged |
| Anthropic data flow | Telemetry, error reporting, and privacy settings |
| Assessment status | Security assessment file exists and review is current |

Results are presented as PASS/WARN/FAIL with a summary table. If a security assessment file exists, the review log is updated automatically.

## What This Checks

This plugin tests three actual risks for local Claude Code setups:

1. **Supply chain** — Is the code running with your credentials auditable and trusted?
2. **Credential storage** — Are your tokens and keys properly protected on disk?
3. **Scope creep** — Do MCP tools have broader access than they need?

Everything else (wire encryption, provider-side security) is handled by TLS and your API providers. This plugin focuses on what you control locally.

## Requirements

- Claude Code
- macOS or Linux
- Git (for local MCP server checks)
- npm or pip (for dependency auditing, optional)

## Recommended Schedule

- **Monthly:** Run `/local-security:local-review`
- **Quarterly:** Review the full security assessment
- **On change:** After adding MCP servers, pulling updates, or changing permissions

## Validation

No test framework exists for Claude Code plugin skills. Run these checks after making changes:

```bash
# From the repo root

# 1. JSON validity
python3 -c "import json; json.load(open('.claude-plugin/marketplace.json')); print('marketplace.json: valid')"
python3 -c "import json; json.load(open('plugins/local-security/.claude-plugin/plugin.json')); print('plugin.json: valid')"

# 2. SKILL.md frontmatter — both skills must have name, description, user-invocable
for skill in local-setup local-review; do
  head -6 "plugins/local-security/skills/$skill/SKILL.md" | grep -q "^name:" && echo "$skill frontmatter: valid" || echo "$skill frontmatter: MISSING"
done

# 3. File structure — all expected files present
for f in .gitignore README.md .claude-plugin/marketplace.json plugins/local-security/.claude-plugin/plugin.json plugins/local-security/.gitignore plugins/local-security/LICENSE plugins/local-security/README.md plugins/local-security/skills/local-setup/SKILL.md plugins/local-security/skills/local-review/SKILL.md; do
  [ -f "$f" ] || echo "MISSING: $f"
done && echo "File structure: complete"
```

After structural checks pass, reinstall and run both skills manually:

```
/plugin uninstall local-security@simplefy-ai
/plugin install local-security@simplefy-ai
```

Restart Claude Code, then run `/local-security:local-setup` and `/local-security:local-review` to verify end-to-end.

## Licence

MIT

## Author

Built by [simplefy.ai](https://simplefy.ai) — Safe. Simple. AI.
