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
- The top-level `VERSION` file is the source of truth (single SemVer line, e.g. `0.1.0`).
- To cut a release: open a PR that bumps `VERSION`. On merge, `.github/workflows/release.yaml` automatically:
  1. Imports a CI signing GPG key
  2. Creates a signed tag `v<VERSION>`
  3. Pushes the tag
  4. Builds per-skill zip artifacts (one zip per `skills/<name>/` folder)
  5. Creates a GitHub release with auto-generated notes from the commit history and attaches the zips
- Tags are `vMAJOR.MINOR.PATCH` (`v0.1.0`, `v0.2.0`, etc.) per [SemVer](https://semver.org). Per-skill versions live in the SKILL.md frontmatter for the moment; repo-level VERSION is the release coordinator.
- The release workflow only fires when `VERSION` itself changes — skill edits without a version bump land on `main` without producing a release. Bump `VERSION` deliberately.

### CI signing key (already configured at org level)

The release workflow signs tags with the ShiftControl-io organization's CI GPG key, which is already provisioned as **organization-level secrets** in the GitHub org:

- `GPG_PRIVATE_KEY` — the CI signing key in ASCII-armor format
- `GPG_PASSPHRASE` — the matching passphrase (empty if the key was generated unprotected)

Both are available to this repo automatically — no per-repo setup needed. The same secrets back the release workflows in other ShiftControl-io repos (e.g. `google_api_sdk`), so the same signing identity appears on tags across the org.

**If you fork this repo** to your own org, you'll need to provision equivalent secrets at your org or repo level — see GitHub's [docs on org-level Actions secrets](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions#creating-secrets-for-an-organization) and the [GPG-import action](https://github.com/crazy-max/ghaction-import-gpg). Without the secrets, the release workflow's tag-import step will fail and no releases will publish; the repo still accepts signed-commit PRs from contributors regardless.

## Code of conduct

Be kind. Disagree on substance, not people. We follow the [Contributor Covenant](https://www.contributor-covenant.org/version/2/1/code_of_conduct/).

## Questions

Open a [discussion](https://github.com/ShiftControl-io/skills/discussions) or [file an issue](https://github.com/ShiftControl-io/skills/issues).
