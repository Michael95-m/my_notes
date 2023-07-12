# Code Quality: Linting and Code Quality

For code quality in python, we should refer to the standard **PEP8**. But following PEP8 standard without proper tools is very time consuming (We can't check all the code everytime when we update and later fix manually)

In this course, we use 

1. **Pylint** as the linting tool
2. **Black** as the formatter
3. **isort** which can sort import statements

## Pylint

In order to perform linting in all files inside a directory, we can run

```bash
pylint --recurisve=y .
```

We can ignore some pylint warning for a function (or) a script by adding some configuration as a comment inside the python code.

```python
class ModelMock:
    # pylint: disable=too-few-public-methods

    def __init__(self, value):
        self.value = value

    def predict(self, X):
        n = len(X)
        return [self.value] * n
```

In the above code, `too-few-public-methods` will be ignored for this class.

Tips: To get a good score with pylint and make a clean code, we should save very long string or dictionary inside a separate file.

## black

We can apply `black` formatter like that

```bash
black .
```

Before applying `black`, we should check the difference between before and after stages.
We can check these differences like
```
black --diff . | less
```

## isort

`isort` can sort import statements by the length of the import statement or alphabetically. 

```bash
isort .
```

## pyproject.toml

We can save the configuration of above tools inside a configuration file named `pyproject.toml`. Below is the example format of pyproject.toml.

```
[tool.pylint.'MESSAGES CONTROL']
disable=[
    "missing-class-docstring",
    "trailing-newlines",
    "missing-function-docstring",
    "missing-module-docstring",
    "invalid-name"
]

[tool.black]
line-length = 88
target-version = ['py39']
skip-string-normalization = true

[tool.isort]
multi_line_output = 3
length_sort = true
```

[Prev](./CloudTest.md) | [Next](./PreCommit-hook.md)