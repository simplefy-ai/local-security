# local-security

A [Claude Code plugin](https://docs.anthropic.com/en/docs/claude-code) for security review and hardening of your local setup.

Two skills that help you understand and maintain the security of your Claude Code environment — MCP servers, credentials, permissions, and data flows.

## Install

```
/plugin marketplace add simplefy-ai/local-security
/plugin install local-security@simplefy-ai
```

## Skills

| Skill | Purpose | Run when |
|-------|---------|----------|
| `/local-security:local-setup` | Discover and document your security posture | Once, or when regenerating |
| `/local-security:local-review` | Validate with automated checks | Monthly, or after any config change |

**Setup** discovers your environment — MCP servers, credentials, privacy settings — and writes a security assessment to `~/.claude/security-assessment.md`. It records what exists but does not flag issues. Previous assessments are archived as dated snapshots.

**Review** validates your setup against 9 checks and populates the mitigation roadmap:

1. Disk encryption
2. MCP server health (discovery + git integrity)
3. Dependency vulnerabilities
4. Credential security (permissions + gitignore)
5. Permissions audit (global/project scope, embedded credentials, accumulated bash permissions)
6. Remote server version pinning
7. Stale credentials
8. Anthropic data flow
9. Assessment status

Results are PASS / WARN / FAIL with a summary table. The review log in your security assessment is updated automatically.

## What it checks

Three actual risks for local Claude Code setups:

1. **Supply chain** — Is the code running with your credentials auditable and trusted?
2. **Credential storage** — Are your tokens and keys properly protected on disk?
3. **Scope creep** — Do MCP tools have broader access than they need?

## Requirements

- Claude Code
- macOS or Linux
- Git (for local MCP server checks)
- npm or pip (for dependency auditing, optional)

## Licence

MIT — see [LICENSE](plugins/local-security/LICENSE).

Built by [simplefy.ai](https://simplefy.ai)
