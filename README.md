# actions

Composite github actions for CICD workflows. These actions are meant to 
reduce the repetitive logic in individual code repositories by providing
functionality that downstream repos can reuse. Actions are organized
by `domain/purpose/action` where `domain` is the programming language
and `purpose` is the intent of the action. By default, actions take the
name `action.yml`, hence the use of a directory structure to differentiate
them.

These actions are not standalone, and are intended to be used in other actions.
For example, as used in `https://github.com/Exabyte-io/periodic-table.js`, a
"consumer" action would look like:

```yaml

...

jobs:
  run-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: Exabyte-io/actions/js/test@main

...

```

where the github action in the `periodic-table.js` repository uses the composite
action `js/test/action.yml` from the `main` branch.






