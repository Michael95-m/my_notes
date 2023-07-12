# Engineering Best Practices

Best Practices which can applied to our codebase include:

- Applying unit test
- Applying integration test
- Applying simulation tests in local for cloud-based services  
- Code quality: linting and formatting
- Git pre-commit hooks
- Makefiles and make 

# Which tools can we use in this process

- For **unit test**, we can use **pytest**
- For Integration test, we will check the whole pipeline is working properly or not. In this case, we can use **[deepdiff](https://pypi.org/project/deepdiff/)** library to compare the output  dictionary result and the expected dictionary result. We can also write some shell scripts to check the working condition of the pipeline.
- To test the code related with cloud in local machine, we can use the cloud simulation tools like **[localstack](https://localstack.cloud/)**.
- For code quality, we can use the tools like pylint, black and isort.
- Pre-commit hook is also useful. This can solve the problem like running linting tools manually every time before committing.
- Makefiles are greate for the automation tasks.


## Not finished yet


