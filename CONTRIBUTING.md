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
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true
git config --global tag.gpgsign true
```

Then add the matching SSH key as a **Signing Key** on GitHub (Settings → SSH and GPG keys → New SSH key → select "Signing Key" as the type).

Verify a commit:

```bash
git log --show-signature -1
# Look for "Good signature" or "Good ... github signature"
```

Other signing options (GPG, S/MIME) work too. See [GitHub's signing guide](https://docs.github.com/en/authentication/managing-commit-signature-verification).

## Adding a new skill

1. Pick a name — lowercase, hyphens, ≤64 chars, no `anthropic` / `claude` reserved words.
2. Create `skills/<name>/SKILL.md` with YAML frontmatter (`name`, `description` ≤200 chars).
3. Optional supporting files under `skills/<name>/references/`.
4. Add an entry to the table in [README.md](README.md).
5. If your skill needs new MCP tools that don't exist yet on `mcp.shiftcontrol.io`, file an issue (or a PR on the MCP server) first.
6. Open a PR. CI validates frontmatter and signed commits. A CODEOWNER reviews.

## Editing an existing skill

Same flow. Bump the version when behavior changes meaningfully (new MCP tool dependency, new workflow step, breaking change to the proposal format). Versions follow [SemVer](https://semver.org).

## Release process

- Merged PRs land on `main`.
- We tag releases as `vMAJOR.MINOR.PATCH` (`v0.1.0`, `v0.2.0`, etc.).
- Each tag triggers a release with zipped skill folders for claude.ai users who can't install from a Git URL directly.

## Code of conduct

Be kind. Disagree on substance, not people. We follow the [Contributor Covenant](https://www.contributor-covenant.org/version/2/1/code_of_conduct/).

## Questions

Open a [discussion](https://github.com/ShiftControl-io/skills/discussions) or [file an issue](https://github.com/ShiftControl-io/skills/issues).
