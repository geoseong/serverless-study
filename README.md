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
  - [Functions with Events](#Functions-with-Events)
  - [X-Ray Tracing](#X-Ray-Tracing)
  - [Plugins](#Plugins)
    - [Serverless-offline plugin](#Serverless-offline-plugin)
    - [tracing plugin: X-Ray](#tracing-plugin-X-Ray)
    - [Dynamodb offline plugin](#Dynamodb-offline-plugin)
    - [pseudo-parameters plugin: CloudFormation Syntax](#pseudo-parameters-plugin-CloudFormation-Syntax)
  - [Resource mgmt](#Resource-mgmt)
    - [DynamoDB mgmt](#DynamoDB-mgmt)
    - [S3 bucket mgmt](#S3-bucket-mgmt)
    - [Cognito mgmt](#Cognito-mgmt)
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
  - [Serverless Documentation: AWS Quick Start](https://serverless.com/framework/docs/providers/aws/guide/quick-start/)
  - [Serverless Framework with Node.js](docs/serverless_framework.md)
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
  - Example (**.aws/credentials**, [참고-AWS Documetation: 명명된 프로필](https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/cli-configure-profiles.html)) 
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

- Configuration
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

- Configuration
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


## Functions with Events
- 참고
  - [Serverless Documentation: AWS - Events](https://serverless.com/framework/docs/providers/aws/guide/events/)
  - [Serverless Documentation: AWS - Functions](https://serverless.com/framework/docs/providers/aws/guide/functions/)

- yml 옵션
  - `functions:` Lambda Functions
  - `events:` [Serverless AWS Lambda Events](https://serverless.com/framework/docs/providers/aws/events/)
    - API Gateway
    - Websocket
    - Kinesis & DynamoDB
    - S3
    - Schedule
    - SNS
    - SQS
    - Application Load Balancer
    - Alexa Skill
    - Alexa Smart Home
    - IoT
    - CloudWatch Event
    - CloudWatch Log
    - Cognito User Pool

- Configuration
  ```yaml
  ## serverless.yml
  functions:  # <- Lambda Functions
    hello:
      handler: handler.hello
      events:
        - schedule: rate(10 minutes)  # <- CloudWatch Event
        - http: # <- API Gateway
            path: hello
            method: get
  ```

- [Enabling CORS](https://serverless.com/framework/docs/providers/aws/events/apigateway#enabling-cors)
  ```yaml
  ## serverless.yml
  provider:
    name: aws
    runtime: nodejs8.10
    stage: ${opt:stage, 'dev'}
    region: ap-northeast-2
    tracing:  # <- X-Ray configuration: 모든 API Gateway와 Lambda functions에 X-ray tracing을 하는 옵션
      apiGateway: true
      lambda: true
    functions:  # <- Lambda Functions
      hello:
        handler: handler.hello
        events:
          - schedule: rate(10 minutes)  # <- CloudWatch Event: 10분마다 반복실행하겠다
          - http: # <- API Gateway Event
              path: hello
              method: get
        cors: true # <- CORS Configuration Default
        # cors: # <- CORS Configuration Advanced
        #   origin: '*'
        #   headers:
        #     - Content-Type
        #     - X-Amz-Date
        #     - Authorization
        #     - X-Api-Key
        #     - X-Amz-Security-Token
        #     - X-Amz-User-Agent
        #   allowCredentials: false
  ```


## X-Ray Tracing
- 참고
  - [Serverless Blog: Serverless Framework v1.41 - X-Ray for API Gateway, Invoke Local with Docker Improvements & More](https://serverless.com/blog/framework-release-v141/)
  - [Serverless Documentation: AWS - Functions -> X-Ray Tracing](https://serverless.com/framework/docs/providers/aws/guide/functions#aws-x-ray-tracing)
  - [Serverless Documentation: AWS - Events -> X-Ray Tracing](https://serverless.com/framework/docs/providers/aws/events/apigateway#aws-x-ray-tracing)
- Configuration
  - 모든 lambda 혹은 apiGateway에 전역으로 설정도 가능하고,
  - 전역설정에 덮어쓰기로 특정 Function에 옵션을 넣는 것도 가능하다
    ```yaml
    provider:  # <- X-Ray configuration: 모든 API Gateway와 Lambda functions에 X-ray tracing을 하는 옵션
      tracing:
        apiGateway: true
        lambda: true
    ```
    or
    ```yaml
    functions:
      hello:
        handler: handler.hello
        tracing: Active   # <- X-Ray configuration: 특정 Function에 X-ray tracing 옵션 적용 가능
      goodbye:
        handler: handler.goodbye
        tracing: PassThrough  # <- X-Ray configuration: 특정 Function에 X-ray tracing 옵션 적용 가능
    ```


## Plugins
### Serverless-offline plugin
- 참고
  - [serverless offline](https://github.com/dherault/serverless-offline)

- Installation
  ```
  $ sls plugin install -n serverless-offline
  ```

- Configuration
  ```yaml
  ## serverless.yml
  custom:
    serverless-offline:
      httpsProtocol: "dev-certs"
      port: 4000

  ...

  plugins:
    - serverless-offline  # <- Plugin install 이후 자동으로 생성되어 있음
  ```

- Running
  ```
  $ sls offline start
  ```

- Tip
  1. VSCode에서 디버그 탭을 들어가서 `launch.json`을 만들어서 디버깅 설정을 한 후
  2. 디버깅하고자 하는 js파일에 breakpoint를 찍고,
  3. 좌측 상단 DEBUG버튼 옆 `재생`버튼을 누르면 하단의 `DEBUG CONSOLE`에서 디버깅 상태를 모니터링 할 수 있으며,
  4. Function 실행 시 breakpoint가 걸리는 것을 확인 할 수 있다.
  
  <p style="margin-left: 1rem;">
    <img src="https://code.visualstudio.com/assets/docs/editor/debugging/debugicon.png" width="100" />
    <img src="https://code.visualstudio.com/assets/docs/editor/debugging/launch-configuration.png" width="300" />
    <img src="https://code.visualstudio.com/assets/docs/editor/debugging/debug-environments.png" width="300" />
  </p>
  
  - `launch.json`
    ```json
    {
      "version": "0.2.0",
      "configurations": [
        {
          "type": "node",
          "request": "launch",
          "name": "sls offline(local)",
          /* "$ which serverless"에서 리턴되는 경로 (macOS기준.) */
          /* "$ DEBUG=true sls offline start" 와 같다. */
          "program": "/Users/${username}/.npm-packages/bin/sls", 
          "args": [
            "offline",
            "start",
          ],
          "env": {
            "DEBUG": "true"
          },
          /* cwd: 해당 스크립트를 실행하는 폴더경로를 지정할 수 있는 옵션이다. */
          /* serverless offline plugin이 설치되어 있는 경로는 ./backend폴더 뿐이므로 backend폴더에서 시작하게 했다. */
          "cwd": "${workspaceFolder}/backend"
        },
      ]
    }
    ```

  <p style="margin-left: 1rem;">
    <img src="https://code.visualstudio.com/assets/docs/editor/debugging/debug-session.png" />
  </p>


### tracing plugin: X-Ray
- 필요 없을 수도...


### Dynamodb offline plugin
  - simulating page


### pseudo-parameters plugin: CloudFormation Syntax


## Resource mgmt
- 참고
  - [Serverless Documentation: AWS - Resources](https://serverless.com/framework/docs/providers/aws/guide/resources)


### DynamoDB mgmt
- 참고
  - [Serverless REST API with DynamoDB and offline support](https://github.com/serverless/examples/tree/master/aws-node-rest-api-with-dynamodb-and-offline)


### S3 bucket mgmt


### Cognito mgmt


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

