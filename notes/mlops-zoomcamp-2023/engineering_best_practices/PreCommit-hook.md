# Precommit hook

We should run all the tools relating with [Linting and Formatting](./Linting_Formatting.md) before commiting. The problem is that we might forget to run these commands sometimes. In order to prevent that, we should have `precommit hooks`.

In order to use precommit hooks, we should install `pre-commit`and create a file named `.pre-commit-config.yaml` by the following procedures.
```
pipenv install --dev pre-commit
pre-commit install
pre-commit sample-config > .pre-commit-config.yaml
```

If you need to clone a fresh repository and pre-commit-config is inside that repository, we can't use immediately. In order to use pre-commit, we should install `pre-commit` first. 

```bash
pipenv install --dev pre-commit
pre-commit install
```

After installation, the configuration inside the pre-commit file is active.

We can add a lot of configuration inside the pre-commit-config file. Below is how we can add isort, black and pylint.


```
repos:
-  repo: https://github.com/pre-commit/pre-commit-hooks
   rev: v3.2.0
   hooks:
     -  id: trailing-whitespace
     -  id: end-of-file-fixer
     -  id: check-yaml
     -  id: check-added-large-files
- repo: https://github.com/psf/black
  rev: 23.3.0
  hooks:
    - id: black
      language_version: python3.9
- repo: https://github.com/pycqa/isort
  rev: 5.12.0
  hooks:
    - id: isort
      name: isort (python)
- repo: local
  hooks:
    - id: pylint
      name: pylint
      entry: pylint
      language: system
      types: [python]
      args: [
          "-rn", # Only display messages
          "-sn", # Don't display the score
          "--recursive=y"
        ]
- repo: local
  hooks:
    - id: pytest-check
      name: pytest-check
      entry: pytest
      language: system
      pass_filenames: false
      always_run: true
      args: [
          "tests/"
        ]
```

[Prev](./Linting_Formatting.md) | [Next](./Makefile.md)