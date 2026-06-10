# AGENTS.md

Guidance for AI coding agents (and humans) working on this repository.

## What this is

A Claude Code plugin: eight markdown skills that guide an external user from zero to a running, exposed app on the Intility Developer Platform. There is no application code — the "source" is SKILL.md prompts, YAML/JSON manifests, and reference docs.

## Layout

```
.claude-plugin/plugin.json       plugin manifest (version is bumped by Release Please — don't edit by hand)
.claude-plugin/marketplace.json  lets this repo serve as its own marketplace
skills/<name>/SKILL.md           one skill per directory
skills/<name>/references/        templates and example conversations, loaded on demand
```

## Conventions

- **Keep SKILL.md under 500 lines.** Move templates, long examples, and walkthroughs to `references/` and link them — they're loaded only when needed.
- **Frontmatter:** `description` is written in third person and includes concrete trigger phrases. `allowed-tools` is the minimum the skill needs — prefer scoped patterns (`Bash(oc delete httproute*)`) over broad ones (`Bash(oc delete*)`); never pre-approve `find`, unscoped `curl`, or `sed` (use Glob/Read/Edit tools instead).
- **Skills detect state, they don't assume it.** Every skill starts by querying (`oc whoami`, `indev cluster list`) rather than trusting conversation memory. Keep it that way.
- **Tone:** plain language for Kubernetes beginners. One command at a time. Introduce a term after showing what it does.
- **Safety rails stay:** `internal` gateway by default, two-step confirmation before `public`, `status` stays read-only, never create a second cluster without explicit double confirmation.

## Before committing

```bash
claude plugin validate . --strict
```

This is also enforced by CI on every PR.

## Commits

Conventional Commits (`fix:`, `feat:`, `docs:`, …) — Release Please derives versions from them. See [CONTRIBUTING.md](CONTRIBUTING.md).
