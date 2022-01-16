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
