


# Main idea

Dependency management is a pain.
How to make it easier for python?

Conda + poetry inside conda.

All non-py dependencies managed by conda.
All py dependencies managed by poetry.

You install your stuff with conda an poetry.
Poetry installs are automatically tracked in pyproject.toml.
Conda installs have to be tracked manually by exporting them to environment.yml.




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


# Poetry

## How poetry .lock and .toml work?

poetry add does:
- adds to pyproject.toml (can specify range of acceptable versions for a dependency)
- finds compatible version, writes specific version in poetry.lock
- installs the dependency with pip

poetry install:
- if poetry.lock satisfies .toml, just install deps at these versions in .lock
- if not, regenerate .lock (probs try to keep old versions and just add stuff. And if conflicting, change old versions.)
- if you deleted .lock, it will actually completely remake it and then install stuff.

## Poetry use:

```bash
poetry add --group dev ruff mypy

# so we also install the dev group
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


















# Conda

## Main conda problem


You install your stuff with conda an poetry.
Poetry installs are automatically tracked in pyproject.toml.
Conda installs have to be tracked manually by exporting them to environment.yml.
Before making a commit, you do:
conda env export > environment.yml

And every time you return to a project, you do:
conda env create -f environment.yml

But, while writing the current commit, you installed some stuff, and you commited changes.
But you forgot to export environment.yml before committing.
Now that commit is no longer atomic, because it has the wrong environment and so doesn't work.

Also, you came back to a project that was mid-commit and had installed stuff.
Now you do env creation and you basically overwrote the active env without the stuff that was installed before.

Also, you come back to a project and forget to first create the env, so you are using the old env.
You add some code, install some conda stuff, export the env.
Now some dependencies don't exist anymore.
Some things are broken - but you don't notice it because you don't have perfect testing set up.

How do we solve these problems?




## Solution



Always paste these 3 lines at once, so there is never any dependency loss:
(or have them be one command - either with a .bashrc alias, or making a conda_update.sh script)

mkdir -p ./.conda
conda env update -f ./.conda/environment.yml    
conda env export > ./.conda/environment.yml
conda env export --from-history > ./.conda/environment_readable.yml


Have this following single-command also be added as a precommit hook script, so you always have atomic working commits,
because this always happens before every commit.

### Explanation of the commands:

conda env update     updates current env with stuff in yml, but doesn't remove anything extra

--from-history only logs stuff that we explicitly installed.
It doesn't also log the dependencies of the stuff we installed.
This makes it nicely readable.


## Making this a precommit hook

Append to .git/hooks/pre-commit
This is the script that git always runs before making a commit.

```bash
cat << 'EOF' >> .git/hooks/pre-commit
#!/bin/sh
conda env update -n myproj -f ./.conda/environment.yml && conda env export -n myproj > ./.conda/environment.yml && conda env export -n myproj --from-history > ./.conda/environment_readable.yml
EOF
```



## Exact bit-by-bit env recreation:

You go rebase to an older commit. Now your env is too new.
How to get it to that correct state?

conda env create -n myproj -f ./.conda/environment.yml

This should do it. 
It Recreates the env exactly as it was when you exported it.

Poetry dependencies are inside conda, so they are also recreated exactly as they were.

But, if you want to be sure, you can also do:
poetry install --sync
to remove any py dependencies that are not in pyproject.toml.



## Alias or script


echo "#!/bin/sh

mkdir -p ./.conda
conda env update -n myproj -f ./.conda/environment.yml
conda env export -n myproj > ./.conda/environment.yml
conda env export -n myproj --from-history > ./.conda/environment_readable.yml

" > conda_update.sh
chmod +x conda_update.sh

Now you always just run:
. /conda_update.sh

Also, you could jsut have these three commands added as an alias in your .bashrc or .zshrc or whatever.
alias cu='conda env update -n myproj -f ./.conda/environment.yml && conda env export -n myproj > ./.conda/environment.yml && conda env export -n myproj --from-history > ./.conda/environment_readable.yml'



