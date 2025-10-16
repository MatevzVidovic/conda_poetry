


# Main idea

Dependency management is a pain.
How to make it easier for python?

Conda + poetry inside conda.

All non-py dependencies managed by conda.
All py dependencies managed by poetry.

You install your stuff with conda an poetry.
Poetry installs are automatically tracked in pyproject.toml.
Conda installs have to be tracked manually by exporting them to environment.yml.


You can make it so conda is activated automatically when you cd into the repo.
We use autoenv for this.

You can make it so conda env is always correctly stored in a file.
We use alias=cosa for this.

You can make it so conda env is always correctly stored in a file before every commit.
We use a precommit hook for this.


# Main useful info


## Setup with autoenv


**Don't forget to change the vars at the top of this script:**
```sh

ENV_NAME=throwaway_env
PY_VERSION=3.12

conda create -n $ENV_NAME python=$PY_VERSION
conda activate $ENV_NAME
pip install poetry
poetry init
poetry config --local virtualenvs.create false # Because our venv will be conda, not a separate regular .venv, like poetry would otherwise use.

# ----- End of basic setup -----

mkdir -p ./.conda
conda env update -f ./.conda/environment.yml    
conda env export > ./.conda/environment.yml
conda env export --from-history > ./.conda/environment_readable.yml


# ----- End of advised conda env file saving setup -----



# ----- Start of autoenv setup -----

cat << EOF >> .env

# ensure conda is available in non-interactive shells
# (only needed if conda init didn’t add this already)
# [ -f "\$HOME/miniconda3/etc/profile.d/conda.sh" ] && . "\$HOME/miniconda3/etc/profile.d/conda.sh"

env_name=$ENV_NAME
conda activate \$env_name

alias pp="poetry run"
alias cosa="conda env update -f ./.conda/environment.yml && conda env export > ./.conda/environment.yml && conda env export --from-history > ./.conda/environment_readable.yml"

EOF

cat << 'EOF' >> .env.leave

conda deactivate

EOF


# ----- Start of making precommit hook -----

cat << 'EOF' >> .git/hooks/pre-commit

#!/bin/sh
conda env update -f ./.conda/environment.yml && conda env export > ./.conda/environment.yml && conda env export --from-history > ./.conda/environment_readable.yml

EOF

```




### If you haven't already, install autoenv:
```sh
# for bash
git clone https://github.com/hyperupcall/autoenv ~/.autoenv
echo 'source ~/.autoenv/activate.sh' >> ~/.bashrc   # or ~/.zshrc
echo 'export AUTOENV_ENABLE_LEAVE=1' >> ~/.bashrc   # or ~/.zshrc
```

or for zsh:
```sh
git clone https://github.com/hyperupcall/autoenv ~/.autoenv
echo 'source ~/.autoenv/activate.sh' >> ~/.zshrc
echo 'export AUTOENV_ENABLE_LEAVE=1' >> ~/.zshrc
```





### Important notes:


#### Rebasing to older commit - exact bit-by-bit env recreation:


You go rebase to an older commit. Now your env is too new.
How to get it to that correct state?

```sh
conda env create -n myproj -f ./.conda/environment.yml
```

This should do it. 
It Recreates the env exactly as it was when you exported it.

Poetry dependencies are inside conda, so they are also recreated exactly as they were.
But, if you want to be sure, you can also do:
```sh
poetry install --sync
```

#### .env.leave can be problematic

If you do:
cd repo_root/code/src
First, repo_root/.env is run, then repo_root/.env.leave,
then repo_root/code/.env, then repo_root/code/.env.leave,
then repo_root/code/src/.env is run.

So if you want the .env scripts to stack, don't have .env.leave scripts in the parent dirs.

But you will probably just be running stuff from repo_root, so this won't be a problem.




### Advice



#### making scripts for poetry aliases

Adding this section to pyproject.toml enables scripts like: 
```
[tool.poetry.scripts]
lint = "ruff check . --fix"
```

poetry run lint    # runs this now

Since we have alias pp="poetry run", we can do:
pp lint





### Basic setup

```sh
ENV_NAME=conda_poetry
PY_VERSION=3.12

conda create -n $ENV_NAME python=$PY_VERSION
conda activate $ENV_NAME
pip install poetry
poetry init
poetry config --local virtualenvs.create false # Because our venv will be conda, not a separate regular .venv, like poetry would otherwise use.
```

### More informational snippets are showed below:
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 



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

```sh
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

For saving the env, updating the env after pulling, creating the env from a file:

Always paste these 3 lines at once, so there is never any dependency loss:
(or have them be one command - either with a .bashrc alias, or making a conda_update.sh script)

```sh
mkdir -p ./.conda
conda env update -f ./.conda/environment.yml    
conda env export > ./.conda/environment.yml
conda env export --from-history > ./.conda/environment_readable.yml
```

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

```sh
cat << 'EOF' >> .git/hooks/pre-commit

#!/bin/sh
conda env update -f ./.conda/environment.yml && conda env export > ./.conda/environment.yml && conda env export --from-history > ./.conda/environment_readable.yml

EOF
```



## Rebasing to older commit - exact bit-by-bit env recreation:

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





# Autoenv functionalities

It is very annoying that every time you go work on the repo, you have to do:
conda activate environment_name

It would be nice if after cd-ing into the repo dir, it got activated automatically.
And, preferably, if after exiting the repo dir it would get deactivated.

direnv is a popular tool that activates on enter and deactivates on exit.
- But it is complicated to make it work with conda.
- And it can't set up aliases.
- It is more meant for .env variable setup and aliases.
It has it's own language, not just running a bash script.
It does so, so you can have one file of instructions, and it knows how to set up on enter and tear-down on exit.



We want tools that just run some script we make on enter, and run another script on exit.

We have smartcd - stores enter and leave scripts outside the repo (~/.smartcd/scripts/...)
Not in repo, so not commited to git - this can be nice in some cases, but generally you don't want this.

But

Autoenv does exactly what we want.
Runs in-repo script on enter (.env script), and runs another in-repo script on exit (.env.leave script).

## Autoenv tool:

### Drawbacks:

Can't rename what files it looks for. So we have to use .env and .env.leave.
And you might want .env to be solely for env vars - they get loaded into your running program.

Also,
cant have scripts inside a directory. So you need both .env and .env.leave in your git root.
This isn't too nice, because it clutters stuff.

But if you have some wrapper dirs in your repo, like repo_root/code/src
then putting the scripts in root doesn't clutter things so badly.
Also, the .env in your src/ is still just holding env vars for your program.

### Installation and setup
```sh
# for bash
git clone https://github.com/hyperupcall/autoenv ~/.autoenv
echo 'source ~/.autoenv/activate.sh' >> ~/.bashrc   # or ~/.zshrc
echo 'export AUTOENV_ENABLE_LEAVE=1' >> ~/.bashrc   # or ~/.zshrc
```

```sh
git clone https://github.com/hyperupcall/autoenv ~/.autoenv
echo 'source ~/.autoenv/activate.sh' >> ~/.zshrc
echo 'export AUTOENV_ENABLE_LEAVE=1' >> ~/.zshrc
```

### .env

```sh
# ensure conda is available in non-interactive shells
# (only needed if conda init didn’t add this already)
# [ -f "$HOME/miniconda3/etc/profile.d/conda.sh" ] && . "$HOME/miniconda3/etc/profile.d/conda.sh"

conda activate myenv
```

### .env.leave

```sh
conda deactivate
```

### Important info on .env.leave

If you will be using a nested structure, like repo_root/code/src,
and you will do "cd repo_root/code/src"
(You might not be doing that, and rather just running things from repo_root - in this case you will be fine)
First, repo_root/.env will be run, then repo_root/.env.leave will be run,
then repo_root/code/.env will be run, then repo_root/code/.env.leave will be run,
then repo_root/code/src/.env will be run.

So if you want the .env scripts to stack, don't have .env.leave scripts in the parent dirs.


### Autoenv tips

- Use absolute-ish paths via the built-ins to avoid breakage in subdirs:
(Autoenv exposes AUTOENV_CUR_FILE and AUTOENV_CUR_DIR for exactly this.)
```sh
# inside any .env
# AUTOENV_CUR_DIR points to the directory that contains THIS .env
source "$AUTOENV_CUR_DIR/venv/bin/activate"
```

- Guard against re-activation if you bounce around:
```sh
if [ "${VIRTUAL_ENV##*/}" != "myenv" ]; then
  . "$AUTOENV_CUR_DIR/.venv/bin/activate"
fi
```


### Autoenv tips





## Alternatives to autoenv

### Smart cd

Stores enter and leave scripts outside the repo (~/.smartcd/scripts/...).

```sh
# install & load
git clone https://github.com/cxreg/smartcd ~/.smartcd-src
cd ~/.smartcd-src && make install
echo 'source ~/bin/load_smartcd' >> ~/.bashrc   # or ~/.zshrc
# create scripts for current directory
smartcd edit enter   # opens your $EDITOR for the "enter" script
smartcd edit leave   # opens your $EDITOR for the "leave" script
```



### zsh-autoenv

Works only with zsh.

But it supports custom enter and leave script names.

Can look up to parent dirs for scripts.

Still can't look to child dirs tho.

```sh
# zsh-autoenv
git clone https://github.com/Tarrasch/zsh-autoenv ~/.zsh-autoenv
source ~/.zsh-autoenv/autoenv.zsh

# custom names and parent lookup
export AUTOENV_FILE_ENTER=".env_enter.zsh"
export AUTOENV_FILE_LEAVE=".env_leave.zsh"
export AUTOENV_HANDLE_LEAVE=1
export AUTOENV_LOOK_UPWARDS=1
```