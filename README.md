# Managing AWS services with 'serverless framework'

## Table of Contents
(- EC2 instance mgmt - VPC plugin usage & Security group mgmt)
- [Managing AWS services with 'serverless framework'](#Managing-AWS-services-with-serverless-framework)
  - [Table of Contents](#Table-of-Contents)
  - [Presentation](#Presentation)
  - [Flowchart](#Flowchart)
  - [References](#References)
- [Tutorials](#Tutorials)
  - [install Serverless Framework and Create project](#install-Serverless-Framework-and-Create-project)
  - [Variable mgmt](#Variable-mgmt)
    - [Recursively reference properties](#Recursively-reference-properties)
    - [Environment variable mgmt](#Environment-variable-mgmt)
  - [Lambda Functions with API Gateway and CloudWatch Events](#Lambda-Functions-with-API-Gateway-and-CloudWatch-Events)
  - [Plugins](#Plugins)
    - [Serverless-offline plugin](#Serverless-offline-plugin)
    - [tracing plugin: X-Ray](#tracing-plugin-X-Ray)
    - [Dynamodb offline plugin](#Dynamodb-offline-plugin)
    - [pseudo-parameters plugin: CloudFormation Syntax](#pseudo-parameters-plugin-CloudFormation-Syntax)
  - [Resource mgmt](#Resource-mgmt)
    - [DynamoDB mgmt](#DynamoDB-mgmt)
    - [S3 bucket mgmt](#S3-bucket-mgmt)
  - [IAM mgmt](#IAM-mgmt)
  - [Lambda Packaging](#Lambda-Packaging)
  - [Lambda Layers](#Lambda-Layers)
  - [Deploying](#Deploying)
  - [View Logging](#View-Logging)
  - [Clearing](#Clearing)
  - [Tips](#Tips)
    - ["CloudFormation way of rollback fail ignore"](#%22CloudFormation-way-of-rollback-fail-ignore%22)


## Presentation
- [SlideShare: AWSKRUG#gudi-serverless framework으로 남몰래 서비스 만들고 지워보기](https://docs.google.com/presentation/d/1yOJMKz0olbGYeiv4j4Y4fZsNqA8me2iHT2WZdmW65M0/edit?usp=sharing)

## Flowchart
(Image from draw.io)

## References
- Introducing serverless framework (2019.06.17 webinar)
  - [Webinar - Getting started with the serverless framework](https://www.youtube.com/watch?v=LXB2Nv9ygQc)
- Serverless Framework: AWS Quick Start
  - [AWS Quick Start](https://serverless.com/framework/docs/providers/aws/guide/quick-start/)
- AWS ARN & NAMESPACE
  - [Amazon 리소스 이름(ARN) 및 AWS 서비스 네임스페이스](https://docs.aws.amazon.com/ko_kr/general/latest/gr/aws-arns-and-namespaces.html)


# Tutorials
## install Serverless Framework and Create project
  ### Serverless framework 설치
  ```
  $ npm install -g serverless
  ```
  ### 프로젝트 생성
  ```
  $ serverless create --template aws-nodejs --path {projectName}
  ```
  ### AWS Credential 확인하기
  - 만약 자신의 PC에 AWS 계정이 여러 개 설정이 되어 있는 경우, 해당 프로젝트를 AWS 에 실제 배포할 때 어떤 AWS 프로필에 배포할 지 프로필명을 알아둔다.
  - 기본적으로는 `default`의 프로필을 사용한다.
  - 경로 ([AWS Documetation: 구성 및 자격 증명 파일](https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/cli-configure-files.html))
    - **macOS** : `/Users/${사용자아이디}/.aws/credentials`
    - **Windows** : `%UserProfile%\.aws/credentials`
  - example (**.aws/credentials**, [참고-AWS Documetation: 명명된 프로필](https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/cli-configure-profiles.html)) 
    ```
    [geoseong] # <- serverless cli의 --aws-profile 플래그의 이름
    aws_access_key_id = #########
    aws_secret_access_key = #########

    [default] # <- serverless cli의 --aws-profile 플래그를 입력 안 하면 이 프로필을 사용하게 된다.
    aws_access_key_id = #########
    aws_secret_access_key = #########
    ```

## Variable mgmt
### Recursively reference properties
- 참고
  - [Serverless Documentation: Recursively reference properties](https://serverless.com/framework/docs/providers/aws/guide/variables#recursively-reference-properties)

- Example
  ```yaml
  ## serverless.yml
  provider:
    name: aws
    stage: ${opt:stage, 'dev'}
    environment:
      MY_SECRET: ${file(../config.${self:provider.stage}.json):CREDS}
  ```
  - `$ sls deploy --stage qa`
  - `stage` is set to **`qa`** from the option supplied to the sls deploy --stage qa command
  - `${self:provider.stage}` resolves to **`qa`** and is used in `${file(../config.${self:provider.stage}.json):CREDS}`
  - `${file(../config.qa.json):CREDS}` is found & the CREDS value is read
  - MY_SECRET value is set


### Environment variable mgmt
- 참고
  - [AWS | Env Variables Example](https://serverless.com/examples/aws-node-env-variables/)

- Example
  ```yaml
  ## serverless.yml
  provider:
    name: aws
    runtime: nodejs8.10
    environment:   # <- 해당 프로젝트 전역 환경변수
      EMAIL_SERVICE_API_KEY: ${file(./env.yml)

  functions:
    createUser:
      handler: handler.createUser
      environment:   # <- 해당 Lambda Function에서만 쓰이는 지역 환경변수
        PASSWORD_ITERATIONS: 4096
        PASSWORD_DERIVED_KEY_LENGTH: 256
  ```
  ```js
  // handler.js
  module.exports.createUser = (event, context, callback) => {
    // logs `4096`
    console.log('PASSWORD_ITERATIONS: ', process.env.PASSWORD_ITERATIONS);
    // logs `256`
    console.log('PASSWORD_DERIVED_KEY_LENGTH: ', process.env.PASSWORD_DERIVED_KEY_LENGTH);
    // logs `KEYEXAMPLE1234`
    console.log('EMAIL_SERVICE_API_KEY: ', process.env.EMAIL_SERVICE_API_KEY);
    const response = {
      statusCode: 200,
      body: JSON.stringify({
        message: 'User created',
      }),
    };

    callback(null, response);
  };
  ```


## Lambda Functions with API Gateway and CloudWatch Events
- 참고
  - [Serverless Documentation: AWS - Events](https://serverless.com/framework/docs/providers/aws/guide/events/)
  - [Serverless Documentation: AWS - Functions](https://serverless.com/framework/docs/providers/aws/guide/functions/)


## Plugins
### Serverless-offline plugin
### tracing plugin: X-Ray
### Dynamodb offline plugin
  - simulating page
### pseudo-parameters plugin: CloudFormation Syntax


## Resource mgmt
### DynamoDB mgmt
- 참고
  - [Serverless REST API with DynamoDB and offline support](https://github.com/serverless/examples/tree/master/aws-node-rest-api-with-dynamodb-and-offline)


### S3 bucket mgmt


## IAM mgmt
- 참고
  - [Serverless Documentation: IAM](https://serverless.com/framework/docs/providers/aws/guide/iam/)


## Lambda Packaging


## Lambda Layers


## Deploying
## View Logging
  - sls logs -f {functionName} --stage {stageName} --aws-profile {profileName}
## Clearing

## Tips
### "CloudFormation way of rollback fail ignore"

