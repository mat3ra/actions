# Actions

Composite github actions for CICD workflows. These actions are meant to
reduce the repetitive logic in individual code repositories by providing
functionality that downstream repos can reuse. Actions are organized
by `domain/purpose/action` where `domain` is the programming language
and `purpose` is the intent of the action. By default, actions take the
name `action.yml`, hence the use of a directory structure to differentiate
them.

### Usage

These actions are not standalone, and are intended to be used in other workflows.
For example, as used in [Periodic Table](https://github.com/mat3ra/periodic-table.js),
a workflow using one of these actions might look like:

```yaml

...

jobs:
  run-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7  # checks out downstream repository
      - uses: actions/checkout@v7  # checks out actions repository
        with:
          repository: mat3ra/actions
          token: ${{ secrets.TOKEN }}
          path: actions

      - uses: ./actions/js/test

...

```

where the workflow in the Periodic Table repository uses the [js/test/action](js/test/action.yml)
from the `main` branch in this repository. In the example, the `actions` repository is cloned into
a relative directory called `actions`. Actions which refer to other actions in this repository
assume this convention and they should be referred to locally as `./actions/.../...`.

### Py/lint usage

Runs [Ruff](https://docs.astral.sh/ruff/) via
[`astral-sh/ruff-action`](https://github.com/astral-sh/ruff-action).

**Default invocation:**

```yaml
- uses: ./actions/py/lint
```

This runs:

```text
ruff check --line-length=120 --target-version=py310
```

(`py310` comes from the default `python-version: '3.10'`.)

**How configuration is resolved**

| Setting          | Source in CI                                        |
|------------------|-----------------------------------------------------|
| Subcommand       | Always `check`                                      |
| `line-length`    | Always `120` (set by the action)                    |
| `target-version` | `python-version` input (default `3.10` → `py310`)   |
| `exclude`        | `exclude` input when set                            |
| Other settings   | `pyproject.toml` / `ruff.toml` when present         |

CLI flags from the action override the same keys in `pyproject.toml` for
`line-length` and `target-version`. Use `pyproject.toml` for advanced lint
configuration.

**Advanced usage** — see [`made/pyproject.toml`](../made/pyproject.toml) for a
full example:

```toml
[tool.ruff]
extend-exclude = [
    "src/js",
    "tests/fixtures",
    "tests/py/unit/fixtures"
]
line-length = 120
target-version = "py310"

[tool.ruff.per-file-ignores]
"__init__.py" = ["F401"]
```

Note: `line-length` and `target-version` in `pyproject.toml` apply locally (e.g.
pre-commit); in CI the action still passes `--line-length=120` and
`--target-version` from `python-version`.

**Optional inputs:**

| Input            | Default   | Description                                           |
|------------------|-----------|-------------------------------------------------------|
| `python-version` | `3.10`    | Maps to Ruff `target-version` (`3.10` → `py310`)    |
| `exclude`        | _(empty)_ | Path pattern passed as `--exclude`                    |
| `ruff-version`   | `0.0.270` | Ruff release to install                               |

```yaml
- uses: ./actions/py/lint
  with:
    python-version: '3.10'
    exclude: some/path
    ruff-version: 0.0.270
```

### Yamllint usage

A github workflow will validate the YAML configuration of all the actions files on every push.
You can set up and run `yamllint` locally with the following:

```bash
virtualenv venv
. venv/bin/activate
pip install yamllint
yamllint .
```
The configuration for `yamllint` is stored in `.yamllint.yml` where we extend the default configuration
with some sensible options for our use case.

### Automated Publishing

Both the `py/publish/action` and the `js/publish/action` will automatically
publish code packages to PyPI and NPM, respectively. The convention for the versions
is determined by the following pseudo-code:
```
date = today's date (YYYY.MM.DD follows semver)
latest_tag = most recent tag in git
if latest_tag starts with date:
    new_tag = latest_tag-(N + 1)
else
    new_tag = date-0
```
There are some useful things to keep in mind when using these publish actions:
1. The `publish` actions follow the philosophy of `setuptools_scm`, where the version
   number is not tracked in git. Instead, it is programmatically generated at time of
   package publication based on the state of the git tree.
2. The logic for versioning JS and python packages is provided by the `git/version/action`.
3. The `new_tag` defined above in `semver` is referred to as a "pre-release" and in `setuptools_scm` is referred
   to as a "post-release". Functionally, there is no difference, but the default convention in `setuptools_scm`
   is to use a `.postN` suffix in place of `-N`. This is configurable and can be controlled in the downstream `pyproject.toml`.
   This means JS packages when published will have a version string of `YYYY.MM.DD-N` and python packages will have
   a version string of `YYYY.MM.DD.postN`.
4. **Important!** For a repository is both a Python AND a JavaScript package, we let the first publish action
   create the git tag and release, and then provide the `publish-tag == 'false'` input parameter
   to the second publish action. This means that the two publish steps in the downstream workflow
   must run sequentially (by use of the `needs` parameter). This is to prevent both actions from
   trying to create the same git tag + release, as the internal logic will likely result in two
   different releases for the same code base!
   - Due to the convention `npm` uses for tracking git changes and ignoring package files,
    define `python` files to be ignored in `js` packages in an `npmignore` file (no leading `.`).
     The default `.gitignore` will also be respected, as well as common workflow files, e.g.
     files in `actions/` and `.github`. Be sure to include `.npmignore` in the `.gitignore`.
     See [this blog post](https://medium.com/@jdxcode/for-the-love-of-god-dont-use-npmignore-f93c08909d8d)
     for justification of this behavior.
5. Make sure the `exabyte-io-bot` has `write` permissions to the repository you're publishing!

### JS release-wip usage

`js/release-wip/action` is **not** a real npm/PyPI publish — it builds a package and
uploads the result as a GitHub **pre-release tarball asset**, tagged after the current
commit, so a not-yet-mergeable WIP commit can be installed by consumers
(`npm install @scope/pkg@<release-asset-url>`) without the package committing its build
output (e.g. `dist/`) to git.

```yaml
- uses: ./actions/js/release-wip
  with:
    package-name: esse
    build-script: transpile-and-build-assets
    github-token: ${{ secrets.BOT_GITHUB_TOKEN }}
```

- The tag/asset name defaults to `wip-<short-commit-sha>` (e.g. `wip-e8ed741`), so every
  commit gets its own immutable tag/asset URL — no cache/integrity headaches for consumers
  from a URL whose content silently changed underneath the same tag. Re-running the
  workflow on the same commit (e.g. a manual re-trigger) updates that commit's
  release/asset in place (`gh release upload ... --clobber`) rather than minting a
  duplicate.
- `build-script` defaults to `transpile`; override it per package (e.g. `esse` needs
  `transpile-and-build-assets`).
- Gate the calling workflow's job on whatever trigger you want (e.g.
  `if: contains(github.event.head_commit.message, '[release]')`) — this action always
  publishes when invoked, it does not decide when to run.

### Notes:

 - Because we opt to not track the version of a published package in git, the `version`
   parameter in `package.json` is unrelated to the state of published code. Please use
   git tags to check out published code.
 - Calling workflows must still use `actions/checkout@v2` before these actions.
   If the repository has `lfs` assets, include `with lfs: true` there.
 - Expression evaluation can be tricky in Github actions. Please see the caveats about
   [Expressions](https://docs.github.com/en/actions/learn-github-actions/expressions#literals).
   - In summary, assume all parameters are strings and when using string literals be sure they
     are only wrapped in single quotes.
 - Workflows that interact directly with Github (like publish actions) avoid
   infinite recursion with downstream workflows triggered on `[push]` because events
   triggered by access tokens do not interact with Github actions. See
   [this page](https://docs.github.com/en/actions/learn-github-actions/events-that-trigger-workflows#triggering-new-workflows-using-a-personal-access-token)
   for details.


### To Do

The `publish` actions automatically push "post-release" patch versions
to their respective package repositories. It also publishes a git tag and
release with the same version. Future work could benefit from downstream
repositories that adhere to the use of commit tools like `commitlint` or
`commitizen` and `semantic-release` to fill in details such as release
notes.
