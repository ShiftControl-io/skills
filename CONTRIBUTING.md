# Contributing to ShiftControl Skills

Thanks for your interest in contributing. This repo holds customer-facing AI skills — instruction sets that pair with the ShiftControl MCP server to drive common workflows.

## Ground rules

1. **Skills propose, humans approve.** Every skill that performs a write MUST present a concrete diff (before → after) and require explicit user approval before calling the MCP server. Never auto-confirm.
2. **No invented identifiers.** Skills MUST source every UUID from a prior `list_*` or `get_*` tool call. Guessing or constructing UUIDs is rejected at review.
3. **Stay within the documented MCP tool surface.** If you need a new tool, file an MCP server change first; don't work around the API.

## Commit signing — required

All commits to `main` MUST be signed. Pull requests with unsigned commits will fail CI and cannot be merged.

Set up signing once on your machine (SSH-based, easiest):

```bash
git config --global gpg.format ssh
git config --global user.signingkey "<path to your public SSH key>"
git config --global commit.gpgsign true
git config --global tag.gpgsign true
```

Replace `<path to your public SSH key>` with the `.pub` file of the SSH key you want to sign with.

Then add the matching SSH key as a **Signing Key** on GitHub (Settings → SSH and GPG keys → New SSH key → select "Signing Key" as the type).

Verify a commit:

```bash
git log --show-signature -1
# Look for "Good signature" or "Good ... github signature"
```

Other signing options (GPG, S/MIME) work too. See [GitHub's signing guide](https://docs.github.com/en/authentication/managing-commit-signature-verification).

## Adding a new skill

1. Pick a name — lowercase, hyphens, ≤64 chars, no `anthropic` / `claude` reserved words.
2. Create `skills/<name>/SKILL.md` with YAML frontmatter: `name`, `description` (≤200 chars), and `metadata.version` (start at `"0.1.0"`, quoted so YAML keeps it a string).
3. Optional supporting files under `skills/<name>/references/`.
4. Add an entry to the table in [README.md](README.md).
5. If your skill needs new MCP tools that don't exist yet on `mcp.shiftcontrol.io`, file an issue (or a PR on the MCP server) first.
6. Open a PR. CI validates frontmatter and signed commits. A CODEOWNER reviews.

Nothing needs adding to `.claude-plugin/`. The repo is published as a single Claude plugin whose root is the repo root, so every folder under `skills/` is picked up automatically.

## Editing an existing skill

Same flow. Bump `metadata.version` in the skill's own frontmatter when behavior changes meaningfully (new MCP tool dependency, new workflow step, breaking change to the proposal format), and update the version in the README catalog table to match. Versions follow [SemVer](https://semver.org), and CI fails the PR if the two disagree.

Each skill versions independently of the plugin. `metadata.version` says what the skill does; the version in `.claude-plugin/plugin.json` coordinates releases. A skill bump without a plugin version bump lands on `main` without cutting a release.

## Release process

- Merged PRs land on `main`.
- To cut a release: open a PR that bumps `version` in `.claude-plugin/plugin.json` and in the plugin entry in `.claude-plugin/marketplace.json`. CI checks that the two match in the same PR. On merge, `.github/workflows/release.yaml` automatically:
  1. Imports a CI signing GPG key
  2. Creates a signed tag for the plugin version
  3. Pushes the tag
  4. Builds per-skill zip artifacts, one zip per `skills/<name>/` folder, each named for that skill's own `metadata.version` rather than the repo version, so a downloaded file says what it holds
  5. Creates a GitHub release with auto-generated notes from the commit history and attaches the zips
- Tags are `vMAJOR.MINOR.PATCH` (`v0.1.0`, `v0.2.0`, etc.) per [SemVer](https://semver.org). Per-skill versions live in SKILL.md frontmatter.
- The release workflow checks for a release when `plugin.json` changes and skips versions that already have one. Claude Code uses the marketplace entry's version to detect updates.

## Code of conduct

Be kind. Disagree on substance, not people. We follow the [Contributor Covenant](https://www.contributor-covenant.org/version/2/1/code_of_conduct/).

## Questions

Open a [discussion](https://github.com/ShiftControl-io/skills/discussions) or [file an issue](https://github.com/ShiftControl-io/skills/issues).
