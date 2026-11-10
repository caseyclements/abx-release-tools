# abx-release-tools

Shared release tooling for ABX (AI Builder Experience) integrations:
`mongodb/strands-mongodb`, `mongodb-developer/ms-agent-framework-mongodb`,
`mongodb-developer/claude-agent-sdk-mongodb-session-store`, and new
integrations that join that list.

POC. Covers the four steps of an ABX release; the
[documentation](docs/index.rst) has the details:

- [Dependency scanning with SilkBomb](docs/scanning.rst)
- [Artifact signing with Garasign](docs/signing.rst)
- [Artifact tracing: Papertrail or the authorized-publication
  report](docs/tracing.rst)
- [Trusted publishing to PyPI](docs/publishing.rst)

## Documentation

Sphinx docs with the Furo theme, under `docs/`:

```
python -m pip install -r docs/requirements.txt
sphinx-build -b html docs docs/_build/html
```

Read the Docs builds the docs from `.readthedocs.yaml`. Update that
file when the repo moves or renames.

## Layout

- `actions/`: GitHub Actions composite actions, consumed with `uses:`.
- `evergreen/`: shell scripts a project's Evergreen config calls, plus the
  thin function templates that call them. Clone this repo at run time and run
  the scripts from the clone, the way the drivers consume
  [drivers-evergreen-tools](https://github.com/mongodb-labs/drivers-evergreen-tools).
  The scripts are shared; the YAML is copied, because Evergreen built-ins such
  as `ec2.assume_role` and `papertrail.trace` have no shell equivalent.
- `scripts/`: shell both CI systems run. `generate-sbom.sh` is called by the
  Evergreen `generate_sbom` function and by the `python/generate-sbom` action,
  so it sits here rather than under `evergreen/`. One copy, so a fix reaches
  both.

## Usage

```
uses: mongodb-labs/abx-release-tools/actions/setup@main
uses: mongodb-labs/abx-release-tools/actions/sbom@main
uses: mongodb-labs/abx-release-tools/actions/gpg-sign@main
uses: mongodb-labs/abx-release-tools/actions/git-sign@main
uses: mongodb-labs/abx-release-tools/actions/authorized-pub@main
uses: mongodb-labs/abx-release-tools/actions/upload-s3-assets@main
uses: mongodb-labs/abx-release-tools/actions/python/generate-sbom@main
uses: mongodb-labs/abx-release-tools/actions/python/publish@main
```

Pin to a commit SHA in real workflows.

The `python/publish` action needs `permissions: id-token: write` on the
publish job. PyPI trusted publishing exchanges the GitHub OIDC token for the
upload; without the permission the publish step fails.

The `sbom`, `gpg-sign`, `git-sign`, `authorized-pub`, and `upload-s3-assets`
actions expect the environment set up by the `setup` action (`RELEASE_ASSETS`,
`S3_ASSETS`, `GARASIGN_ENVFILE`, `KONDUKTO_ENVFILE`, `AWS_BUCKET`). Run
`setup` first.

`python/generate-sbom` needs no `setup`: nothing in it touches AWS. It writes
the `sbom.json` that `sbom` then reads, so run it before `sbom`. Name the
project type if the repo keeps its dependencies anywhere but
`requirements.txt`; see [the scanning docs](docs/scanning.rst).

To trace a release from GitHub Actions, `authorized-pub` writes
`authorized-publication.txt` and `upload-s3-assets` puts it in the release S3
bucket. Run the two actions in that order: there is no `python/post-publish`
release chain here (see Out of scope).

## Provenance

The GitHub Actions were extracted from
[mongodb-labs/drivers-github-tools](https://github.com/mongodb-labs/drivers-github-tools)
at commit `99b1cbd729bc83bf7fc74e1b2a97fffad3d65d7b`, then hardened. Each file
carries a provenance comment listing what diverges and why.

One action is not extracted: `python/generate-sbom`. Upstream has no such
action, because upstream leaves making the input SBOM to the caller. The
Evergreen path here wraps that step, so the two CI systems disagreed on who
owns it. Its file says so.

Action names and outputs are unchanged. Two behaviours changed deliberately,
both recorded under "Input contract" below. Nothing changed silently: the rule
stays that semantics track upstream so fixes cherry-pick, and where this repo
leaves upstream it says so in the file and here.

Upstream, these same actions run in production in pymongo's release
workflow
([release-python.yml](https://github.com/mongodb/mongo-python-driver/blob/main/.github/workflows/release-python.yml)),
which pins the same commit the actions were extracted from: `setup` and
`gpg-sign` in the post-publish chain, `authorized-pub` and `sbom` through
`full-report`, `upload-s3-assets` in post-publish, and `git-sign` through
pre-publish's `tag-version`. The hardening above means the bytes this repo
ships are ahead of upstream; the upstream fixes are tracked in
[PYTHON-6234](https://jira.mongodb.org/browse/PYTHON-6234).

The Evergreen scripts and templates follow public, working Evergreen release
pipelines. Each file cites its source:

- `evergreen/sign_artifacts.yml`: mongodb/mongo-rust-driver
  (`.evergreen/releases.yml`, `.evergreen/release-sign.sh`).
- `evergreen/silkbomb.yml`: mongodb/mongo-csharp-driver
  (`evergreen/evergreen.yml`, `evergreen/upload-sbom.sh`). `upload` only; the
  augmented SBOM that ships with a release comes from the GitHub Actions
  `sbom` action.
- `evergreen/papertrail.yml`: mongodb/mongo-java-driver (`.evergreen/.evg.yml`)
  and mongodb/mongo-rust-driver (`.evergreen/releases.yml`).

The clone-and-call-the-scripts arrangement follows
[drivers-evergreen-tools](https://github.com/mongodb-labs/drivers-evergreen-tools),
which pymongo and the Rust driver consume that way: `include:` entries in both
projects are same-repo paths, neither declares an Evergreen module, and the
shared repo arrives as a clone at `$DRIVERS_TOOLS` whose scripts the config
calls.

The scripts carry the same fixes as the actions: the credential env file is
truncated not appended, written under `umask 077`, and rejected if a value
holds a newline; `gpg` is pinned with `--local-user`; file names reach the
container as positional arguments rather than as script text; and every
expansion that reaches a command line is quoted.

## Hardening

The extracted actions have defects. Two sources found them, kept distinct
because they carry different weight.

**Five came from `mongodb-developer/claude-agent-sdk-mongodb-session-store`**,
the first ABX repo to run Garasign signing in production. It carries local
rewrites of `setup` and `gpg-sign` rather than using the extracted versions,
for exactly these reasons. Each has been met in a real release; they are folded
back here so the next integration needs no copy of its own:

- **`gpg-sign` word-split the file list.** `for filename in ${{ inputs.filenames }}`
  splits on whitespace, so `my package.tar.gz` became `my` and
  `package.tar.gz`, neither of which exists. Nothing was signed for it and the
  step exited 0, leaving a release that looks signed. The list now reaches the
  container NUL-delimited over stdin.
- **`gpg-sign` did not pin the key.** Without `--local-user "$GPG_KEY_ID"`, a
  Garasign account holding more than one key signs with whichever gpg picks,
  and the signature verifies against a key the release notes do not name.
- **Credentials were written to a docker `--env-file` unchecked.** The format
  is line-oriented, so a newline in a password silently truncated it and the
  run failed as a Garasign auth error, which is a misleading place to look.
  Presence
  and newlines are now checked first, under `umask 077`.
- **`setup.sh` reconstructed the secret-name prefix** to strip it, which cannot
  work for an ARN. The secret is now loaded under a blank alias, so keys arrive
  unprefixed, and a comma in the secret id is rejected before it is read.
- **`setup.sh` used bare `mkdir`**, so a rerun of a job failed.

**The rest were found auditing the extracted actions while applying those
five.** These are reasoned from the code rather than observed in a release,
which is a weaker claim; each is nonetheless reachable from an input a caller
supplies:

- **Inputs were interpolated into `run:` bodies** in `gpg-sign`,
  `authorized-pub` and `upload-s3-assets`. In `authorized-pub`, a GitHub token
  was among them. `${{ }}` is substituted before bash parses the script, so
  an input arrives as syntax rather than as a value. Inputs now arrive through
  `env:`. `git-sign` still interpolates its `command`, because running an
  arbitrary git command is what it is for; callers must not pass it untrusted
  input.
- **A pattern matching nothing was not an error.** Every action that takes
  `filenames` now fails on one. An upload or signing step that quietly handles
  zero files and exits 0 is the failure these steps exist to catch.
- **A file matched by two patterns was handled twice.** `dist/*.whl dist/*`
  signed, reported and uploaded the same artifact twice. In `gpg-sign` the
  second pass then looked for a signature the first pass had already moved, so
  a correct release failed reporting that no signature was produced, which is
  the message that is supposed to mean the container produced nothing.
- **`sbom` and `git-sign` left expansions unquoted**, including
  `--sbom-in /repo/${SBOM_IN_PATH}`, which a caller supplies. A path containing
  a space asked silkbomb for two files; one containing a `*` asked it for
  whatever matched in the working directory.
- **`setup.sh` wrote to `/tmp`.** It now writes under `RUNNER_TEMP`, which the
  runner scopes to the job and empties at the end of it. `/tmp` stays as the
  fallback off a runner.

These are generic fixes, so they go upstream to drivers-github-tools too,
rather than living only here.

## Input contract

Two behaviours changed. Both are additive for a correct caller and breaking
for one that was relying on the old silence.

Every action taking `filenames` accepts a **newline-separated** list as well as
the original space-separated one, and treats the input as newline-separated
whenever it contains a newline:

```yaml
filenames: dist/*.whl dist/*.tar.gz      # space-separated, as upstream
filenames: |                             # newline-separated, one pattern per line
  dist/*.whl
  my package.tar.gz
```

The newline form is the only way to name a file containing a space. A
space-separated list cannot express one, which is the defect that started this.

**A pattern matching nothing is now an error**, where upstream exited 0. A
caller relying on a glob that sometimes matches nothing must stop passing it,
rather than expect the step to shrug.

## Tests

`pytest` (`pip install -r tests/requirements.txt`). `.github/workflows/test.yml`
runs it, and `pre-commit --hook-stage manual`, on every push and pull request.

Nearly all of the logic here lives in the `run:` bodies of composite actions,
which neither existing check opened: the pre-commit shellcheck hook matches
`.sh` files, and a `run:` body is not one. So the suite parses each
`action.yml` and tests those bodies directly.

The static tests assert that no `${{ }}` survives into a `run:` body, that
every expression names a declared input, that every script sets
`-euo pipefail` and declares `shell: bash`, and that each body passes
shellcheck at its default severity. The behavioural tests run the extracted
bodies against a fixture directory with `docker`, `gh` and `aws` stubbed on
`PATH`.

The fixture artifact is named `my package.tar.gz`. Every file-list defect
listed above turned on a name containing a space, including two introduced
while fixing the first one.

The suite pulls no container and makes no AWS call, so `gpgloader`,
`--local-user`, the silkbomb invocations and the secret load are not exercised
by it. A real release is still the first test of those.

## Out of scope (deliberately)

The drivers-github-tools release pipeline around `python/post-publish`
(`full-report`, `bump-version`, `tag-version`, `compliance-report`) is not
extracted. ABX repos do not use the Evergreen +
S3 release pipeline those actions assume. ABX repos never use
drivers-github-tools directly: when a repo needs one of those actions, extract
it from drivers-github-tools into this repo first, then use it from here.

## Maintenance rules

- This repo is ABX-owned. ABX re-pins action SHAs, fixes lint findings, and
  adapts when DevProd changes SilkBomb, Garasign, or Papertrail interfaces.
- Generic fixes go upstream to drivers-github-tools too, so drift stays
  visible instead of silent.
- Run `pre-commit run --all-files` before committing, and `pytest`.
