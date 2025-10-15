

## Setup

conda create -n ruff_mypy python=3.12
conda activate ruff_mypy
pip install poetry

poetry init

poetry add --group dev ruff mypy





### Poetry alias scripts (if not using make):

[tool.poetry.scripts]
lint = "ruff check . --fix"
format = "ruff format ."
typecheck = "mypy ."

Just run:
(set bash alias pp="poetry run")
poetry run lint
poetry run typecheck










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




# Solution


Have this following single-command also be added as a precommit hook script, so you always have atomic working commits:

Always paste these 3 lines at once, so there is never any dependency loss:
(or have them be one command - either with a .bashrc alias, or making a conda_update.sh script)

conda env update -n myproj -f ./.conda/environment.yml    
conda env export -n myproj --from-history > ./.conda/environment.yml
conda env export -n myproj --from-history > ./.conda/environment_readable.yml




conda env update     updates current env with stuff in yml, but doesn't remove anything extra

--from-history only logs stuff that we explicitly installed.
It doesn't also log the dependencies of the stuff we installed.
This makes it nicely readable.


## Making this a precommit hook

cat << 'EOF' >> .git/hooks/pre-commit
#!/bin/sh
conda env update -n myproj -f environment.yml && conda env export -n myproj --from-history > environment.yml && conda env export -n myproj --from-history > environment_readable.yml
EOF




## Exact bit-by-bit env recreation:
You go rebase to an older commit. Now your env is too new.
How to get it to that correct state?


conda env create -n myproj -f environment.yml
Should do it. It Recreates the env exactly as it was when you exported it.

Poetry dependencies are inside conda, so they are also recreated exactly as they were.
But, if you want to be sure, you can also do:
poetry install --sync
to remove any py dependencies that are not in pyproject.toml.



## Alias or script


echo "#!/bin/sh

conda env update -n myproj -f environment.yml    
conda env export -n myproj --from-history > environment.yml
conda env export -n myproj --from-history > environment_readable.yml

" > conda_update.sh
chmod +x conda_update.sh

Now you always just run:
. /conda_update.sh

Also, you could jsut have these three commands added as an alias in your .bashrc or .zshrc or whatever.
alias cu='conda env update -n myproj -f environment.yml && conda env export -n myproj --from-history > environment.yml && conda env export -n myproj --from-history > environment_readable.yml'



