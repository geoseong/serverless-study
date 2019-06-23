# Serverless Framework with Node.js
- [References: AWS Quick Start](https://serverless.com/framework/docs/providers/aws/guide/quick-start/)

## Create a new Serverless Service/Project
```
$ serverless create --template aws-nodejs --path {my-service}
```
or
```
$ mkdir {projectName}
$ cd {projectName}
$ serverless create --template aws-nodejs
```

## Change into the newly created directory
```
$ cd my-service
```

## Deploy, test and diagnose your service
### Deploy the Service
```
$ serverless deploy -v
```

### Deploy the Function
```
$ serverless deploy function -f hello
```

### Invoke the Function
Invokes a Function and returns logs.
```
$ serverless invoke -f hello -l
```

### Fetch the Function Logs
Open up a separate tab in your console, set your Provider Credentials and stream all logs for a specific Function using this command.
```
$ serverless logs -f hello -t
```

## Cleanup
```
$ serverless remove
```