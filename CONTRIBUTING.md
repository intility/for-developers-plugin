# Contributing

Thanks for helping make the Developer Platform plugin better. It's an early alpha — feedback and small fixes are exactly what it needs.

## Found a bug? Have a wish?

Open an [issue](https://github.com/intility/ext-devplatform-plugin/issues). Include what you asked Claude to do, what it tried (the failing command is the best clue), and the cluster name if relevant.

## Making changes

1. The plugin is markdown and JSON — skills live in `skills/<name>/SKILL.md`, with longer material in `skills/<name>/references/`.
2. Validate before committing:

   ```bash
   claude plugin validate . --strict
   ```

3. To try your changes live, add your local checkout as a marketplace in Claude Code:

   ```
   /plugin marketplace add /path/to/ext-devplatform-plugin
   /plugin install ext-devplatform-plugin@intility
   ```

4. Open a PR against `main`. The `validate` check must pass before merging.

## Commit messages

Use [Conventional Commits](https://www.conventionalcommits.org/) — releases are automated with Release Please, which derives the next version from commit messages:

- `fix: …` → patch release
- `feat: …` → minor release
- `feat!: …` or `BREAKING CHANGE:` → major release
- `docs:`, `chore:`, `ci:` → no release

## Releases

Release Please opens (and keeps updating) a release PR as changes land on `main`. Merging that PR bumps `version` in `.claude-plugin/plugin.json`, updates `CHANGELOG.md`, and tags a GitHub Release.

**Why the version bump matters:** installed users only receive plugin updates when the `version` field changes — merging the release PR is what actually ships your change. Don't edit the version by hand.
