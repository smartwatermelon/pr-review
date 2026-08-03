# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

This repository *is* a single Claude Code Skill, packaged as an installable plugin: `SKILL.md` at the repo root defines "pr-review," a skill that does a deep-dive review of a teammate's GitHub pull request — tracing claims against the real repo instead of reasoning from the diff alone, and staging findings as a pending GitHub review rather than posting them directly. `SKILL.md`'s content is the entire functional deliverable; everything else in the repo exists to distribute and validate that one file.

- `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` make `/plugin marketplace add smartwatermelon/pr-review` work in Claude Code. `plugin.json`'s `skills` field points at the repo root (`./`), since `SKILL.md` lives there rather than under the conventional `skills/<name>/` subdirectory; without it, the plugin loads and the skill still works, but Claude Desktop's plugin detail panel won't list it under "Skills." Keep this field if `SKILL.md`'s location ever changes.
- `.github/workflows/validate.yml` runs `scripts/validate_skill.py` on push/PR: checks `SKILL.md`'s frontmatter has required keys and no non-portable ones (`compatibility`, `allowed-tools` break plugin-marketplace validation), and that the plugin/marketplace manifests are valid JSON.
- `README.md` and `LICENSE` are independently written for this repo.

The `.claude/` directory (note: no hyphen, different from `.claude-plugin/`) is boilerplate from a local git template (project-specific config/hook scaffolding for Andrew's global Claude Code infrastructure at `~/.claude/`). Nothing in it is customized for this repo.

## Andrew vs. "the user" — read this before editing SKILL.md

Andrew Rich (smartwatermelon) is this repo's author and maintainer — that's who's editing `SKILL.md` when working in *this* repo. But `SKILL.md`'s content is consumed by anyone who installs the plugin and runs it in their own project, with their own agent. Inside `SKILL.md`, "the user" always means that installer/invoker, not Andrew specifically — the skill text should never assume the reader is Andrew, reference his personal repos/preferences, or bake in review criteria that only make sense for his projects. When Andrew wants to record his own personal review preferences (as opposed to a generic improvement to the skill), that belongs in his own `~/.claude` memory or global config, not in this repo's `SKILL.md`.

## Working on this repo

There is no build step. The only thing to run locally is the validator:

```bash
python3 scripts/validate_skill.py
```

- Keep `SKILL.md` generic and portable — it's read by other people's agents, working on other people's repos, not just Andrew's.
- The skill deliberately keeps the human as an explicit checkpoint before anything gets posted, merged, or submitted (Phase 5 and 6). Don't loosen those gates when adding capability.
- If you bump behavior meaningfully, bump `metadata.version` in `SKILL.md`'s frontmatter and `version` in `.claude-plugin/plugin.json` together — `SKILL.md` itself nudges installers to compare that version against the repo when they suspect their copy is stale.
