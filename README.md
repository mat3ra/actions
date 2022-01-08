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
For example, as used in [Periodic Table](https://github.com/Exabyte-io/periodic-table.js),
a workflow using one of these actions might look like:

```yaml

...

jobs:
  run-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2  # checks out downstream repository
      - uses: actions/checkout@v2  # checks out actions repository
        with:
          repository: Exabyte-io/actions
          token: ${{ secrets.TOKEN }}
          path: actions

      - uses: ./actions/js/test

...

```

where the workflow in the Periodic Table repository uses the [js/test/action](js/test/action.yml)
from the `main` branch in this repository. In the example, the `actions` repository is cloned into
a relative directory called `actions`. Actions which refer to other actions in this repository
assume this convention and they should be referred to locally as `./actions/.../...`.

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
    new_tag = latest_tag-N + 1
else
    new_tag = date-0
```
There are some useful things to keep in mind when using these publish actions:
1. In the `js/publish/action`, the `npm version` command partially provides this logic,
which actually creates a new commit on the branch from
which we are publishing, along with the new git tag.
2. For python-only packages, the logic in `git/version/action` fills in the missing
behavior provided by `npm version` to generate git tags, (without creating a commit).
3. For a repository is both a Python AND a JavaScript package, we let `js/publish/action`
run first to create the commit and tag, and then call `py/publish/action` with `publish-tag == 'false'`
4. Make sure the `exabyte-io-bot` has write permissions to the repository you're publishing!

### Notes:

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
 - `js/publish/action.yml` will automatically version bump when publishing releases
   to both NPM and Github. It follows a date-based prerelease convention.
 -  `py/publish/action.yml` mimics `js/publish` when publishing releases to PyPI and Github.
    It follows the same date-based pre-release convention (post-release in `setuptools_scm` parlance).
    

### To Do

The `js/publish` action will automatically patch version bump when publishing to NPM.
It will also push the new version tag and release to Github.
Future work could benefit from downstream repositories that adhere to the use of
commit tools like `commitlint` or `commitizen` and `semantic-release` to fill in details
such as release notes.

The content below describes an ideal setup where Github branch policies work as described.
Unfortunately that doesn't appear to be the case at this time.

The exabyte bot should be configured to be the only user that can force push to a matching branch.

- **Important** In order for the "main" branch to be kept in-sync with the commits
  that may happen during publication, the downstream repository must have
  `Allow force pushes` enabled in the `Branch protection rule` for the branch to be
  published. This allows commits generated by, for example, `npm`, to be pushed back
  to the protected branch so that there is no confusion about the state of versioning
  on the "main" branch.
   - NB: "main" branch refers to whichever branch is reserved for publication, and could
     be `main`, `master`, `dev`, etc., depending on the calling action
