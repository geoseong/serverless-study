# Managing AWS services with 'serverless framework'

## Table of Contents
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
    - [serverless-vpc-plugin](#serverless-vpc-plugin)
  - [Resource mgmt](#Resource-mgmt)
    - [DynamoDB mgmt](#DynamoDB-mgmt)
    - [S3 bucket mgmt](#S3-bucket-mgmt)
    - [Cognito mgmt](#Cognito-mgmt)
  - [IAM mgmt](#IAM-mgmt)
  - [Lambda Packaging](#Lambda-Packaging)
    - [CLI command](#CLI-command)
    - [yml Configuration](#yml-Configuration)
  - [Lambda Layers](#Lambda-Layers)
  - [Deploying](#Deploying)
  - [View Logging](#View-Logging)
  - [Clearing](#Clearing)
  - [Tips](#Tips)
    - [CloudFormation: UPDATE_ROLLBACK_FAILED](#CloudFormation-UPDATEROLLBACKFAILED)


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
        - schedule: rate(10 minutes)  # <- CloudWatch Event: 10분마다 반복실행하겠다
        - http: # <- API Gateway Event
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
- [VPC Configuration](https://serverless.com/framework/docs/providers/aws/guide/functions#vpc-configuration)
  ```yaml
  # serverless.yml
  service: service-name
  provider: aws

  functions:
    hello:
      handler: handler.hello
      vpc:
        securityGroupIds:
          - securityGroupId1
          - securityGroupId2
        subnetIds:
          - subnetId1
          - subnetId2
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
    - **`program`**:
      - `$ which serverless`에서 리턴되는 경로 (macOS기준.), `$ DEBUG=true sls offline start` 와 같다.
    - **`cwd`**:
      - 해당 스크립트를 실행하는 폴더경로를 지정할 수 있는 옵션이다.
      - serverless offline plugin이 설치되어 있는 경로는 ./backend폴더 뿐이므로 backend폴더에서 시작하게 했다. */
          
    ```json
    {
      "version": "0.2.0",
      "configurations": [
        {
          "type": "node",
          "request": "launch",
          "name": "sls offline(local)",
          "program": "/Users/${username}/.npm-packages/bin/sls", 
          "args": [
            "offline",
            "start",
          ],
          "env": {
            "DEBUG": "true"
          },
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


### serverless-vpc-plugin
- [smoketurner/serverless-vpc-plugin](https://github.com/smoketurner/serverless-vpc-plugin#readme)
- // TODO: 다른 functions, Resources와 함께 구성 시 deploy할 때마다 VPC를 새로 만드는 지 확인을 해 볼 것
```yaml
# add in your serverless.yml

plugins:
  - serverless-vpc-plugin

provider:
  # you do not need to provide the "vpc" section as this plugin will populate it automatically
  vpc:
    securityGroupIds:
      -  # plugin will add LambdaExecutionSecurityGroup to this list
    subnetIds:
      -  # plugin will add the "Application" subnets to this list

custom:
  vpcConfig:
    cidrBlock: '10.0.0.0/16'

    # if createNatGateway is a boolean "true", a NAT Gateway and EIP will be provisioned in each zone
    # if createNatGateway is a number, that number of NAT Gateways will be provisioned
    createNatGateway: 2

    # When enabled, the DB subnet will only be accessible from the Application subnet
    # Both the Public and Application subnets will be accessible from 0.0.0.0/0
    createNetworkAcl: false

    # Whether to create the DB subnet
    createDbSubnet: true

    # Whether to enable VPC flow logging to an S3 bucket
    createFlowLogs: false

    # Whether to create a bastion host
    createBastionHost: false
    bastionHostKeyName: MyKey # required if creating a bastion host

    # Whether to create a NAT instance
    createNatInstance: false

    # Optionally specify AZs (defaults to auto-discover all availabile AZs)
    zones:
      - us-east-1a
      - us-east-1b
      - us-east-1c

    # By default, S3 and DynamoDB endpoints will be available within the VPC
    # see https://docs.aws.amazon.com/vpc/latest/userguide/vpc-endpoints.html
    # for a list of available service endpoints to provision within the VPC
    # (varies per region)
    services:
      - kms
      - secretsmanager

    # Optionally specify subnet groups to create. If not provided, subnet groups
    # for RDS, Redshift, ElasticCache and DAX will be provisioned.
    subnetGroups:
      - rds
```

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
- 참고
  - [Serverless Documentation: Packaging](https://serverless.com/framework/docs/providers/aws/guide/packaging/)
### CLI command
AWS Lambda function에 배포될 파일구조를 미리 볼 수 있다.
```
$ serverless package
```
- `.serverless` 폴더 안에 배포될 파일들이 저장된다

```
$ serverless package --package done -> `done`폴더 안에 배포 파일들 저장
$ serverless package --package done/isaid -> `done/isaid`폴더 안에 배포 파일들 저장
```
- `--package` 플래그를 붙이면 사용자가 직접 경로를 지정할 수 있다.

### yml Configuration
- **`include`** / **`exclude`**
  - 배포 시 포함되어야 할 것과 포함되어야 하지 말아야 할 것을 지정할 수 있다.
  - **exclude 안 해도 기본적으로 제외되는 파일 리스트들**
    - .git/**
    - .gitignore
    - .DS_Store
    - npm-debug.log
    - .serverless/**
    - .serverless_plugins/**
  ```yaml
  package:
    exclude:  # node_modules 하위폴더를 제외하지만 node_modules/node-fetch/ 는 다시 포함시킨다
      - node_modules/**
      - '!node_modules/node-fetch/**'
  ```
  ```yaml
  package:  # src 하위폴더를 제외하고 src/function/handler.js은 포함시킨다.
    exclude:
      - src/**
    include:
      - src/function/handler.js
  ```
- **`individually`**
  - 기본적으로는 yml파일에 구성되어 있는 function 전체가 통으로 옵션이 먹지만,
  - `individually: true` 옵션으로 function 개별 설정도 가능하다.
    - 가장 상위에 두면 모든 function들이 개별설정하도록 적용
    - functions 안에 두면
  ```yaml
  package:
    individually: true
    exclude:
      - functions/**

  functions:
    hello:
      handler: functions/hello.index
      package:
        include:
          - functions/hello.js
    bye:
      handler: functions/bye.index
      package:
        include:
          - functions/bye.js
  ```
  ```yaml
  functions:
    hello:
      handler: functions/hello.
    bye:
      handler: functions/bye.index
      package:
        individually: true
  ```
- **`excludeDevDependencies`**
    - devDependency가 제외 되는것을 원치 않을 때 사용
    ```yaml
    package:
      excludeDevDependencies: false
    ```
- **`artifact`**
  - 이미 packaging이 되어 있는 다른 package를 사용하고자 할 때 사용
  - 이 옵션이 설정되어 있으면 배포 시 별도로 packaging단계를 거치지 않는다
  - AWS S3의 경로도 지정 가능함
    - [ ] 그러나 실패하고 있음
  - 실습을 위해 `$ cd backend && sls package --package done` 을 해서 `done/hello.zip` 압축파일을 만든다.
  - 전역으로 설정하는 방법
  ```yaml
  package:
    artifact: done/hello.zip
  ```
  - 개별 설정하는 방법
  ```yaml
  package:
    individually: true

  functions:
    hello:
      handler: functions/hello.index  # hello.zip안의 functions폴더 안에 있는 hello.js
    package:
      artifact: done/hello.zip
    events:
      - http:
          path: hello
          method: get
  ```
  - 참고
    - Lambda의 코드 파일들은 `/var/task` 폴더 안에 저장되어 있음.

## Lambda Layers


## Deploying
- 참고
  - [Serverless Documentation: Deploying](https://serverless.com/framework/docs/providers/aws/guide/deploying/)
- 작동원리
  1. CloudFormation 스택이 생성되지 않았다면, S3 버킷을 새로 생성해서 그 안에 소스코드들이 압축된 zip파일을 넣는다.
  2. 배포 시 기존에 배포되어있는 내용과 로컬의 배포될 내용이 같다면 배포절차를 중단한다.
  3. Zip files of your Functions' code are uploaded to your Code S3 Bucket.
  4. 정의된 IAM Roles, Functions, Events and Resources들이 AWS CloudFormation template으로 추가된다
  5. The CloudFormation Stack 동명의 새로운 template으로 업데이트된다.
  6. function들은 배포될때마다 새로운 버전이 생긴다.
- `serverless.yml`에 설정된 것 모두 배포하기
  ```
  $ serverless deploy
  ```
- `--verbose`: 배포 시 CloudFormation Stack에서 출력하는 이벤트를 확인하고 싶다면..
  ```
  $ serverless deploy --verbose
  ```
- `--stage`, `--region` 플래그로 stage명과 region변경 가능
  ```
  $ serverless deploy --stage production --region eu-central-1
  ```
- `function --function`: 특정 function만 지정해서 개별배포 가능
  ```
  $ serverless deploy function --function myFunction
  ```
- `--aws-profile`: 내 로컬에 복수개의 AWS profile이 있다면, `.aws/credentials`에 있는 profile 이름을 지정하여 배포하고자 하는 계정 지정 가능
  ```
  $ serverless deploy deploy --aws-profile myProfile
  ```
- `--package`
  - 배포 대상을 `$ serverless package`를 통해 packaging된 폴더 경로를 지정해서 배포
  ```
  $ serverless deploy --package path-to-package
  ```

## View Logging
  ```
  sls logs -f {functionName} --stage {stageName} --aws-profile {profileName}
  ```

## Clearing
- 참고
  - [Serverless Documentation: Removal](https://serverless.com/framework/docs/providers/aws/guide/services#removal)
  ```
  $ serverless remove -v
  ```

## Tips
### CloudFormation: UPDATE_ROLLBACK_FAILED
> 문제가 되는 리소스의 Logical ID를 입력해서 무시하자
- **문제발견**
  - `$ serverless deploy` 중에 `UPDATE_ROLLBACK_FAILED` Status가 나오는 경우 발견
- **원인**
  - 존재하지 않는 Lambda Layer 버전을 Lambda에서 참고하려고 해서 에러가 뜨는 문제였는데
	- 실패하고 있는 Stack에서 `Resources` 탭을 누르면, 표가 나온다.
  	- [이미지첨부]
	- 나오는 표에서 `Logical ID`라고 되어 있는 부분을 유심히 본다.
    - [이미지첨부]
	- 그래서 잘못된 Lambda Layer를 바라보는 Lambda Function에 해당되는 `Logical ID` 혹은 다수개의 Logical ID들을 모아서 **`콤마(,)`를 중간에 붙여** 리스트를 완성했다.
		- Ex) FrontendRootLambdaFunction,FrontendAdminLambdaFunction
	- 이제 상단의 `Actions`메뉴를 펼친 뒤 `Coutinue Update Rollback`을 누른다.
  	- [이미지첨부]
	- 그럼 모달창이 뜨는데, 거기서 `Advanced`를 눌러서 펼쳐본다.
  	- [이미지첨부]
	- 펼쳐서 나오는 텍스트입력란에 `Logical ID`들을 콤마로 붙인 리스트를 붙여넣고 `Continue(?였나?)` 버튼을 누르면 그에 해당하는 리소스들은 **update rollback 리스트에서 제거**되면서 `UPDATE_ROLLBACK_COMPLETE` 가 뜬 것을 확인 할 수 있었다.
