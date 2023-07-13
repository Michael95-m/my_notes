# Make and Makefile

`Makefile` is good for automation purpose. It can automate most of our building and testing, and much more. We can define and automate specific stages in `makefile`. But you want to use `make`, you need to install `make`  first (In linux, make is already installed).

Below is the sample Makefile used in the course.

```
LOCAL_TAG:=$(shell date +"%Y-%m-%d-%H-%M")
LOCAL_IMAGE_NAME:=stream-model-duration:${LOCAL_TAG}

test:
	pytest tests/

quality_checks:
	isort .
	black .
	pylint --recursive=y .

build: quality_checks test
	docker build -t ${LOCAL_IMAGE_NAME} .

integration_test: build
	LOCAL_IMAGE_NAME=${LOCAL_IMAGE_NAME}  bash integration-test/run.sh

publish: build integration_test
	LOCAL_IMAGE_NAME=${LOCAL_IMAGE_NAME} bash scripts/publish.sh

setup:
	pipenv install --dev
	pre-commit install
```

In this example makefile, `build` stage is dependent on `quality_checks` and `test`. If one of these two stages fail, build stage won't be run. `Integration_test` stage is dependent on `build` stage.

We can run various stages in the makefile like that. Eg. if you want to run `quality_check` stage, you can type 
```make quality_check``` in the terminal. For `setup`, you can type ```make type``` in the terminal.

[Prev](./PreCommit-hook.md)