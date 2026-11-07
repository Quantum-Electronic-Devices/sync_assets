# Sync statics

A composite GitHub Action that keeps static files in a repository in sync with
the assets of the latest release of another repository. When a new release is
published, it copies the release's PDF assets over the files with the same name
in this repository, records the new version, and opens a pull request.

It uses only `git` and the `gh` CLI that are preinstalled on GitHub-hosted
runners. The pull request itself is created by
[`create-pull-request`](https://github.com/Quantum-Electronic-Devices/create-pull-request).

## How it works

1. **Validate and read the versions.** The action checks that `repository` looks
   like `<owner>/<repo>` and that `version_file` is a relative path to an
   existing file inside the repository. It then asks GitHub for the latest
   release of the source repository (`releases/latest`, which excludes drafts
   and pre-releases) and reads the currently synced tag from `version_file`.
2. **Stop if up to date.** If both tags are equal, the remaining steps are
   skipped.
3. **Copy the assets.** The release's `*.pdf` assets are downloaded to a
   temporary directory outside the workspace. Every file in the repository
   whose **name** is exactly the name of a downloaded asset is overwritten with
   it, wherever it lives in the repository. Hidden files and directories
   (anything with a path component starting with `.`) are never touched. An
   asset that matches no file produces a warning. The new tag is then written
   to `version_file`.
4. **Open a pull request** with the changes, on the branch
   `update_statics/<source-repo-name>/<tag>`, titled
   `Bump <owner>/<repo> from <old tag> to <new tag>`, with the label
   `dependencies`.

The action is idempotent. If a pull request for that branch already exists in
any state (open, merged or declined), it does nothing, so it is safe to run on a
daily schedule.

## Requirements

The calling job must:

1. Check out the repository, preferably with `persist-credentials: false`.
2. Grant these permissions:

   ```yaml
   permissions:
     contents: write
     pull-requests: write
   ```

3. Provide a version file that already exists in the repository and is tracked
   by git. It contains the tag that is currently synced (only the first 100
   bytes are read, and only the characters `A-Za-z0-9._+-` are kept). It may be
   empty on the first run.
4. Have the "Allow GitHub Actions to create and approve pull requests"
   repository setting enabled (Settings → Actions → General) if you use the
   default `GITHUB_TOKEN` as `github-token`.
5. Have a `dependencies` label in the repository if you want the pull request
   labelled. A missing label only produces a warning.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `repository` | yes | | The repository to sync, as `<owner>/<repo>`. |
| `version_file` | yes | | Path of the file that stores the synced version, relative to the repository root. Must not contain `..`. |
| `remote-token` | yes | | Token with read access to the releases of the source repository. Used to look up and download the release. |
| `github-token` | yes | | Token that can push a branch and create a pull request in this repository. |
| `commit-user-name` | no | `github-actions[bot]` | Login of the account the commit is attributed to (`<app-slug>[bot]` for a GitHub App). |
| `commit-user-id` | no | `41898282` | Numeric user ID of that account. The default is the ID of `github-actions[bot]` on github.com. Look it up with `gh api users/<login> --jq .id` (for an app: `gh api "users/<app-slug>[bot]" --jq .id`). It is used to build the commit email `<id>+<login>@users.noreply.github.com`, which links the commit to the account. |

The action has no outputs.

## Usage

```yaml
name: Sync statics

on:
  schedule:
    - cron: "0 6 * * *"
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false

      - uses: OWNER/REPO@v1
        with:
          repository: my-org/documentation
          version_file: statics/VERSION
          remote-token: ${{ secrets.DOCS_READ_TOKEN }}
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

Replace `OWNER/REPO@v1` with the repository that hosts this action and pin a tag
or commit SHA.

## Example

Suppose `my-org/documentation` publishes release `v1.4.0` with the assets
`manual.pdf` and `datasheet.pdf`, and the repository contains
`statics/manual.pdf`, `statics/VERSION` (currently `v1.3.0`) but no
`datasheet.pdf`. The action will:

- overwrite `statics/manual.pdf` with the new asset,
- warn that `datasheet.pdf` matches no file in the repository,
- write `v1.4.0` to `statics/VERSION`,
- open the pull request "Bump my-org/documentation from v1.3.0 to v1.4.0" from
  the branch `update_statics/documentation/v1.4.0`.

## Notes and limitations

- **Only PDF assets are synced.** The download pattern is fixed to `*.pdf`.
  Other asset types are ignored.
- **Matching is by file name only.** If the repository contains several files
  with the same name in different directories, all of them are overwritten.
  Asset names are compared as exact strings and are never used as globs.
- **Only tracked files end up in the pull request.** Files that already exist in
  the repository are updated; new files are never added. If no asset matches
  anything, the pull request still contains the version file change.
- **Release tags are restricted.** The latest tag must match
  `^[A-Za-z0-9][A-Za-z0-9._+-]*$`, because it is used in a branch name, a commit
  message and a pull request title. Any other tag makes the action fail.
- **Private source repositories.** `remote-token` needs permission to read
  releases and download assets of the source repository. The default
  `GITHUB_TOKEN` cannot do that for another private repository.
- **CI on the pull request.** Pull requests created with the default
  `GITHUB_TOKEN` do not trigger other workflows. To have your checks run on
  them, pass a personal access token or a GitHub App token as `github-token`
  (and set `commit-user-name` and `commit-user-id` to that bot's login and ID).
