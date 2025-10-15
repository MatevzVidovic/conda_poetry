


# Idea

Dependency management is a pain.
How to make it easier for python?

Conda + poetry inside conda.

All non-py dependencies managed by conda.
All py dependencies managed by poetry.

You install your stuff with conda an poetry.
Poetry installs are automatically tracked in pyproject.toml.
Conda installs have to be tracked manually by exporting them to environment.yml.
Before making a commit, you do: 

But, while making the current commit, you install some stuff and forget to

## Setup

```bash
ENV_NAME=conda_poetry
PY_VERSION=3.12

conda create -n $ENV_NAME python=$PY_VERSION
conda activate $ENV_NAME
pip install poetry
poetry init
poetry config --local virtualenvs.create false # Because our venv will be conda, not a separate regular .venv, like poetry would otherwise use.
```

## How poetry .lock and .toml work?

poetry add does:
- adds to pyproject.toml (can specify range of acceptable versions for a dependency)
- finds compatible version, writes specific version in poetry.lock
- installs the dependency with pip

poetry install:
- if poetry.lock satisfies .toml, just install deps at these versions in .lock
- if not, regenerate .lock (probs try to keep old versions and just add stuff. And if conflicting, change old versions.)
- if you deleted .lock, it will actually completely remake it and then install stuff.

## Use:

```bash
poetry add --group dev ruff mypy

# install only n
poetry install --with dev
```


## Poetry alias scripts (useful if not using makefile):
(have: alias pp="poetry run" so this is faster to type)

To pyproject.toml, add sth like this to define your aliases:

[tool.poetry.scripts]
lint = "ruff check . --fix"
format = "ruff format ."
typecheck = "mypy ."

Then later you just run:
poetry run lint
poetry run typecheck

or, if you have the suggested alias:
pp lint
pp typecheck
