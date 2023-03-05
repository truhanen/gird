[//]: # (This README.md is autogenerated from README_template.md with the script
         render_readme.py)

# Gird

Gird is a lightweight & general-purpose [Make][make]-like build tool & task
runner for Python.

[make]: https://en.wikipedia.org/wiki/Make_(software)

Gird can be used to manage any project where some tasks need to be executed
automatically when some dependencies are updated. The goal of Gird is to combine
the following features.

- A simple, expressive, and intuitive rule definition and execution scheme very
  close to that of Make.
- Configuration in Python, allowing straightforward and familiar usage, without
  the need for a dedicated rule definition syntax.
- Ability to take advantage of Python's flexibility and possibility to easily
  integrate with Python libraries and tools.
- Emphasis on API simplicity & ease of use.

## Installation

Install Gird from PyPI with `pip install gird`, or from sources with
`pip install .`.

### Requirements

Gird requires Python version 3.8 or newer, and is supported on Linux & macOS.

Gird also requires [`make`][make] to be available on the command line. It should
be available on all Linux distributions via the default package manager, and on
macOS via Xcode. Most implementations of Make will do, as long as they support
the `.PHONY` special target.

## Usage

Define "rules" in *girdfile.py*. Depending on the composition of a rule
definition, a rule can, for example,

- define a recipe to run a task, e.g., to update a target file,
- define prerequisites for the target, such as dependency files or other rules,
  and
- use Python functions for more complex dependency & recipe functionality.

A rule is invoked by `gird {target}`, with optional parallel execution. To list
all rules, run `gird list`.

### Example girdfile.py

This is the girdfile.py of the project itself.

```python
from itertools import chain
from pathlib import Path

from gird import Phony, rule
from scripts import assert_readme_updated, get_wheel_path, render_readme

WHEEL_PATH = get_wheel_path()

RULE_PYTEST = rule(
    target=Phony("pytest"),
    recipe="pytest -n auto",
    help="Run pytest.",
)

RULE_CHECK_FORMATTING = rule(
    target=Phony("check_formatting"),
    recipe=[
        "black --check gird scripts test girdfile.py",
        "isort --check gird scripts test girdfile.py",
    ],
    help="Check formatting with Black & isort.",
)

RULE_CHECK_README_UPDATED = rule(
    target=Phony("check_readme_updated"),
    recipe=assert_readme_updated,
    help="Check that README.md is updated based on README_template.md.",
)

RULES_TEST = [
    RULE_PYTEST,
    RULE_CHECK_FORMATTING,
    RULE_CHECK_README_UPDATED,
]

rule(
    target=Phony("test"),
    deps=RULES_TEST,
    help="\n".join(f"- {rule.help}" for rule in RULES_TEST),
)

rule(
    target=Path("README.md"),
    deps=chain(
        *(Path(path).iterdir() for path in ("scripts", "gird")),
        [Path("girdfile.py"), Path("pyproject.toml")],
    ),
    recipe=render_readme,
    help="Render README.md based on README_template.md.",
)

rule(
    target=WHEEL_PATH,
    recipe="poetry build --format wheel",
    help="Build distribution packages for the current version.",
)

rule(
    target=Phony("publish"),
    deps=WHEEL_PATH,
    recipe=f"twine upload --repository gird {WHEEL_PATH}",
    help="Publish packages of the current version to PyPI.",
)
```

Respective output from `gird list`:

```
pytest
    Run pytest.
check_formatting
    Check formatting with Black & isort.
check_readme_updated
    Check that README.md is updated based on README_template.md.
test
    - Run pytest.
    - Check formatting with Black & isort.
    - Check that README.md is updated based on README_template.md.
README.md
    Render README.md based on README_template.md.
dist/gird-1.5.0-py3-none-any.whl
    Build distribution packages for the current version.
publish
    Publish packages of the current version to PyPI.
```

### Example rules

A rule with files as its target & dependency. When the rule is invoked, the
recipe is executed only if the dependency file has been or will be updated,
or if the target file doesn't exist.

```python
import pathlib
import gird
WHEEL = pathlib.Path("package.whl")

RULE_BUILD = gird.rule(
    target=WHEEL,
    deps=pathlib.Path("module.py"),
    recipe="python -m build --wheel",
)
```

A rule with a phony target (not a file). The rule is always executed when
invoked.

```python
RULE_TEST = gird.rule(
    target=gird.Phony("test"),
    deps=WHEEL,
    recipe="pytest",
)
```

A rule with other rules as dependencies, to group multiple rules together,
and to set the order of execution between rules.

```python
gird.rule(
    target=gird.Phony("all"),
    deps=[
        RULE_TEST,
        RULE_BUILD,
    ],
)
```

A rule with a Python function recipe.

```python
import json
JSON1 = pathlib.Path("file1.json")
JSON2 = pathlib.Path("file2.json")

def create_target():
     JSON2.write_text(
         json.dumps(
             json.loads(
                 JSON1.read_text()
             ).update(value2="value2")
         )
     )

gird.rule(
    target=JSON2,
    deps=JSON1,
    recipe=create_target,
)
```

A Python function as a dependency to arbitrarily trigger rules. Below, have
a local file re-fetched if a remote version is updated.

```python
@gird.dep
def is_remote_newer():
    return get_timestamp_local() < get_timestamp_remote()

gird.rule(
    target=JSON1,
    deps=is_remote_newer,
    recipe=fetch_remote,
)
```

Compound recipes for, e.g., setup & teardown. All subrecipes of a rule are
run in a single shell instance.

```python
gird.rule(
    target=JSON2,
    deps=JSON1,
    recipe=[
        "export VALUE2=value2",
        create_target,
        "unset VALUE2",
    ],
)
```

Define rules in a loop, or however you like.

```python
RULES = [
    gird.rule(
        target=source.with_suffix(".json.gz"),
        deps=source,
        recipe=f"gzip -k {source.resolve()}",
    )
    for source in [JSON1, JSON2]
]

```

## Implementation of Gird

Internally, Gird generates Makefiles & uses Make to run tasks, but interacting
with Make in any way isn't obligatory when using Gird. In the future, Make as a
dependency of Gird might be replaced altogether.
