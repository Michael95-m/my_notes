# Testing cloud functions

To test the function related with cloud services, we use the tools named `localstack`. We run the localstack from the  `docker compose`. Below is the content of the dockerfile related with `localstack`.

```
services:
  kinesis:
    image: localstack/localstack
    ports:
      - "4566:4566"
    environment:
      - SERVICES=kinesis
```

In this dockerfile, we create a service named `kinesis` which will simulate aws kinesis.

And running command to create resources for localstack from `aws cli` is almost the same as the real command to create resources.

The command to check the list of kinesis streams in our cloud.

```
aws kinesis list-streams
```

With localstack, we need to add extra parameter named `--endpoint-url=http://localhost:4566` to simulate the function.

To simulate the command above with the localstack, 

```
aws kinesis --endpoint-url=http://localhost:4566 list-streams
```

Note: We need to up docker compose before using localstack.

