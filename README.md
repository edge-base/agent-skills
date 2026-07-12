# EdgeBase Agent Skills

Installable EdgeBase AI skill bundles generated from the main EdgeBase monorepo.

If you reached this repository from search, start with the official AI guide first:

- https://edgebase.fun/docs/getting-started/ai

## Contents

- `skills/edgebase` — the single public EdgeBase skill with generated SDK and CLI references

## Install

- Codex: copy `skills/edgebase` into `$CODEX_HOME/skills/edgebase`
- GitHub-backed skill installers: use the `edge-base/agent-skills` repository and install the `edgebase` skill from `skills/edgebase`

## What This Skill Does

- detects EdgeBase tasks from natural language and repo context
- routes to the narrowest CLI or SDK reference by runtime and trust boundary
- helps agents avoid mixing browser/mobile client code with admin/server-only SDKs

## Source Of Truth

This distribution is generated from:

- `skills/edgebase/SKILL.md`
- `skills/edgebase/agents/openai.yaml`
- `skills/edgebase/references/generated/*.md`
- the leaf `llms.txt` files in the main EdgeBase repository

Do not edit the generated files in the published repo by hand. Update the source repo and resync instead.
