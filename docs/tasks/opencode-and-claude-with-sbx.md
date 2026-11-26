---
id: opencode-and-claude-with-sbx
status: done
priority: high
kind: task
branch: task/opencode-and-claude-with-sbx
---

# opencode and claude with sbx

- add instructions on how to install sbx and opencode
- add instruction on how to intall sbx and claude

## How to review

This change is documentation only, so there are no unit tests to run.

1. Read `docs/SbxAgents.md` from top to bottom. It covers installing sbx per OS,
   `sbx login` and the network policy, then a Claude Code section and an OpenCode
   section.
2. Check the commands against the Docker pages each section links to. They were
   copied from those pages in November 2026. The guide is Windows-only. Two
   commands were translated from bash to PowerShell because the docs show only
   bash: piping `$env:ANTHROPIC_API_KEY` into `sbx secret set anthropic`, and the
   OpenCode Zen `sbx secret set-custom` command. Check those two by hand.
3. Not run: none of these commands were executed. The work was done inside an sbx
   sandbox, where `sbx` is not installed. To check end to end on Windows, run
   `winget install -h Docker.sbx`, then `sbx login`, then `sbx run claude` and
   `sbx run opencode` in a test project. Each agent's TUI should start in the
   sandbox.
