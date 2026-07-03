# versionbump

Example repo that keeps the version in `composer.json` in sync with the branch name — automatically, on release branches.

Open a branch named `release/X.Y.Z`, push it, and a GitHub Action writes that version into `composer.json` for you. No manual version edits, no forgetting to bump.

## How it works

Two small workflows:

### 1. Bump version — [`.github/workflows/bump-version.yml`](.github/workflows/bump-version.yml)

Runs on every push to a `release/**` branch.

- Reads the version from the branch name (`release/1.2.3` → `1.2.3`).
- Fails if the name isn't `release/X.Y.Z` (semver, no prefix).
- Writes it into `composer.json` **only if it differs** from the current value, then commits and pushes back to the branch.

The "only if it differs" check is what stops the Action from re-triggering itself in a loop.

### 2. Enforce branch — [`.github/workflows/enforce-branch.yml`](.github/workflows/enforce-branch.yml)

Runs on every pull request into `main`.

- Fails if the PR's source branch isn't `release/X.Y.Z`.
- Combined with a required status check, this means **`main` can only be updated from a release branch** — so a release can't accidentally skip the version bump.

GitHub branch rules can't restrict the source branch of a PR on their own, so this needs both the workflow **and** a required status check (see setup below).

## Conventions

- Branch name: **`release/X.Y.Z`** — semver, no `v` prefix.
- The version lives in **`composer.json`** (the `version` field).
- `main` is only updated through pull requests.

## Behaviour

| You do | What happens |
| --- | --- |
| Push `release/0.1.0` (composer at `0.0.0`) | Version bumped to `0.1.0`, committed to the branch |
| Push `release/0.1.0` (composer already `0.1.0`) | Nothing — no commit, no loop |
| Push `release/0.1.0` with a wrong version by hand | Corrected to `0.1.0` — the branch name wins |
| Push `release/foo` | Workflow fails — name must be `release/X.Y.Z` |
| Open a PR into `main` from `release/0.1.0` | Enforce check passes, merge allowed |
| Open a PR into `main` from any other branch | Enforce check fails, merge blocked |

## Using this in another repo

**Requirements:**

- A `composer.json` at the root, with the version stored there (not in code).
- The team follows the `release/X.Y.Z` naming (semver, no prefix).
- `main` is protected with pull requests; direct push is off.
- `release/*` branches don't block the bot's push.
- Actions write permission is on (Settings → Actions → General → Workflow permissions).

**Steps:**

1. Copy both workflow files into `.github/workflows/`.
2. Merge them into `main` via a PR.
3. Make the enforce check required: Settings → Rules → your `main` ruleset → **Require status checks to pass** → add **`check-branch`**.
4. Verify with a test `release/x.y.z`: the version should bump, and `main` should only accept merges from `release/*`.

## Notes

- The bump commit is made with `GITHUB_TOKEN`, which **does not trigger other workflows**. If something downstream (CI, build, deploy) needs to run after the bump, use a PAT or a GitHub App token.
- If a repo keeps its version in code instead of `composer.json` (e.g. a `VERSION` constant), adapt the bump step first.
- If a repo doesn't use `release/*` branches yet, the release process needs that step added before this can work.
- Renaming the `check-branch` job later breaks the required status check binding — update the ruleset to match if you rename it.
