

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



