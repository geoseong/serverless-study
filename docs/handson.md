# Table of Contents

- [Table of Contents](#table-of-contents)
  - [Reference](#reference)
  - [Create Serverless Framework Project](#create-serverless-framework-project)
  - [Upload static content](#upload-static-content)
  - [Enable static website hosting](#enable-static-website-hosting)
  - [User management](#user-management)

## Reference

- [Aws Lambda, Amazon Api Gateway, S3, Dynamodb And Cognito Example](https://github.com/andreivmaksimov/serverless-framework-aws-lambda-amazon-api-gateway-s3-dynamodb-and-cognito)
  - [Serverless Framework - Building Web App Using AWS Lambda, Amazon API Gateway S3 DynamoDB And Cognito - Part-1](https://hands-on.cloud/serverless-framework-building-web-app-using-aws-lambda-amazon-api-gateway-s-3-dynamo-db-and-cognito-part-1/)
  - [Serverless Framework - Building Web App Using AWS Lambda, Amazon API Gateway S3 DynamoDB And Cognito - Part-2](https://hands-on.cloud/serverless-framework-building-web-app-using-aws-lambda-amazon-api-gateway-s-3-dynamo-db-and-cognito-part-2/)

## Create Serverless Framework Project

- serverless framework로 프로젝트 생성

  ```sh
  $ mkdir wild-rides-serverless-demo
  $ cd wild-rides-serverless-demo
  $ sls create -t aws-nodejs -n wild-rides-serverless-demo

  # 'wild-rides-serverless-demo' 폴더 안에 'serverless.yml', 'handler.js', '.gitignore' 생성됨
  ```

- npm script를 위하여 npm 프로젝트를 init한다

  ```sh
  $ npm init -y

  # package.json 생성됨
  ```

- `serverless.yml`에 **custom 변수**, **S3 Resource** 추가

  ```yaml
  # S3 bucket name은 global하게 unique한 이름을 가져가야 하므로 [내이름] 영역에 자신의 이름을 겹치지 않게 입력한다.
  custom:
    bucketName: gudi-serverless-handson-[내이름]
  
  ...

  resources:
    Resources:
      GudiSlsStorage:
        Type: AWS::S3::Bucket
        Properties:
          BucketName: ${self:custom.bucketName}
  ```

- S3 bucket 배포

  - 로컬환경에 단일 프로필만 존재한다면...
  
    ```sh
    # create s3 bucket
    $ sls deploy --verbose
    ```

  - 로컬환경에 다수개의 aws profile이 있고, default가 아닌 다른 profile에 적용하고자 한다면...

    ```sh
    # create s3 bucket
    $ sls deploy --aws-profile ["aws프로필명"] --verbose
    ```

## Upload static content

- 프론트 페이지를 `awslabs/aws-serverless-workshops`로 부터 다운받아 온다

  ```sh
  $ git clone https://github.com/awslabs/aws-serverless-workshops/

  # 'aws-serverless-workshops' 폴더가 생성되면서 그 안에 리포지토리 파일들이 들어가 있음
  ```

- 내가 배포했던 S3 Bucket에다가 `WebApplication/1_StaticWebHosting/website` 폴더만 업로드한다
  
  - 로컬환경에 단일 프로필만 존재한다면...

    ```sh
    # 나의 bucket에 파일 업로드. 업로드 잘 되었는지 확인할 것

    $ aws s3 sync ./aws-serverless-workshops/WebApplication/1_StaticWebHosting/website s3://gudi-serverless-handson-[내이름]
    ```

  - 로컬환경에 다수개의 aws profile이 있고, default가 아닌 다른 profile에 적용하고자 한다면...

    ```sh
    # 나의 bucket에 파일 업로드. 업로드 잘 되었는지 확인할 것
    # [AWS프로필명]: OSX(~/.aws/credentials), Windows(`%UserProfile%\.aws/credentials`)

    $ aws s3 sync ./aws-serverless-workshops/WebApplication/1_StaticWebHosting/website s3://gudi-serverless-handson-[내이름] --profile [AWS프로필명]
    ```

- 더이상 `aws-serverless-workshops` 폴더는 필요 없으니 지운다

  ```sh
  # 'aws-serverless-workshops' 폴더 삭제
  $ rm -Rf ./aws-serverless-workshops
  ```

- configuration에 Bucket Policy를 추가한다.

  ```yaml
    resources:
      Resources:
        ...
        GudiSlsStoragePolicy:
          Type: AWS::S3::BucketPolicy
          Properties:
            Bucket: ${self:custom.bucketName}
            PolicyDocument:
              Statement:
                - Effect: 'Allow'
                  Principal: '*'
                  Action:
                    - 's3:GetObject'
                  Resource:
                    - 'arn:aws:s3:::${self:custom.bucketName}/*'
  ```

- 배포하기 전에 자주 사용하는 스크립트들은 package.json에 미리 작성해 놓도록 하자
  - 로컬환경에 단일 프로필만 존재한다면...

    ```json
    {
      "name": "wild-rides-serverless-demo",
      "version": "1.0.0",
      "description": "",
      "main": "handler.js",
      "scripts": {
        "deploy": "sls deploy --verbose",
        "remove": "sls remove --verbose",
        "package": "sls package"
      },
      "keywords": [],
      "author": "",
      "license": "ISC"
    }
    ```

  - 로컬환경에 다수개의 aws profile이 있고, default가 아닌 다른 profile에 적용하고자 한다면...

    ```json
      {
        "name": "wild-rides-serverless-demo",
        "version": "1.0.0",
        "description": "",
        "main": "handler.js",
        "scripts": {
          "deploy": "sls deploy --aws-profile [~/.aws/credentials 안의 적용하고자 하는 프로필명] --verbose",
          "remove": "sls remove --aws-profile [~/.aws/credentials 안의 적용하고자 하는 프로필명. default만 있다면 --aws-profile 없어도 됨] --verbose",
          "package": "sls package"
        },
        "keywords": [],
        "author": "",
        "license": "ISC"
      }
      ```

- 미리 작성 해 놓은 **npm 스크립트**를 이용하여 배포하기

  ```sh
  # serverless.yml의 변경사항 배포
  $ npm run deploy
  ```

## Enable static website hosting

- `serverless.yml`의 `resources` -> `Resources` -> `GudiSlsStorage` -> `Properties` 안에 S3 웹사이트 호스팅 설정관련 섹션인 `WebsiteConfiguration` 을 추가한다
- 또한 `resources` 안에 `Website URL`정보를 담게 하는 `Outputs:` 섹션을 추가한다
  > `Outputs`이란, 해당 CloudFormation Stack 접근 시 확인하고 싶은 정보를 Customize할 수 있다. 다른 스택으로 가져오거나(교차 스택 참조를 생성하기 위해), 응답으로 반환하거나(스택 호출을 설명하기 위해), 또는 AWS CloudFormation 콘솔에서 볼 수 있는 출력 값을 선언할 수 있다.

  ```yaml
  resources:
    Resources:
      GudiSlsStorage:
        Type: AWS::S3::Bucket
        Properties:
          BucketName: ${self:custom.bucketName}
          WebsiteConfiguration:
            IndexDocument: index.html
    Outputs:
      GudiSlsStorageURL:
        Description: "Gudi Serverlss framework handson Bucket Website URL"
        Value:
          "Fn::GetAtt": [ GudiSlsStorage, WebsiteURL ]
  ```

- 변경사항 배포 후 `Stack Outputs` 을 확인 해 본다.

  ```sh
  $ npm run deploy

  Stack Outputs
  GudiSlsStorageURL: http://gudi-serverless-[내이름].s3-website.ap-northeast-2.amazonaws.com
  ```

## User management

- 회원관리를 위한 `AWS Cognito User Pool`을 생성
  - `serverless.yml` -> `resources:` -> `Resources:`에 아래 섹션을 추가하고 배포한다.
  - 아래 추가할 Cognito `Type`과 `Properties`는 CloudFormation 문법인데, `AutoVerifiedAttributes: ['email']` 속성을 넣어 주어야지만 호스팅 페이지에서 회원가입 시 인증코드가 이메일로 보내는 것이 가능하다.
    - 참고문서: [`CloudFormation -> AWS::Cognito::UserPool`](https://docs.aws.amazon.com/ko_kr/AWSCloudFormation/latest/UserGuide/aws-resource-cognito-userpool.html#cfn-cognito-userpool-autoverifiedattributes)

    ```yaml
    GudiSlsUserPool:
      Type: AWS::Cognito::UserPool
      Properties:
        UserPoolName: GudiSls
        AutoVerifiedAttributes: ['email']
    ```

    ```sh
    $ npm run deploy

    # 배포 후 AWS Cognito -> Manage User Pools를 들어가서 'GudiSls' pool이 생성된 것을 확인한다.
    ```

- App client 생성
  - 마찬가지로 `serverless.yml` -> `resources:` -> `Resources:`에 아래 섹션을 추가하고 배포한다.
  > Author: "I do not understand, why they call it App Client in web console"

    ```yaml
    GudiSlsUserPoolClient:
      Type: AWS::Cognito::UserPoolClient
      Properties:
        ClientName: GudiSlsWebApp
        GenerateSecret: false
        UserPoolId:
          Ref: 'GudiSlsUserPool'
    ```

    ```sh
    $ npm run deploy

    # 배포 후 AWS Cognito -> Manage User Pools -> 'GudiSls' pool 클릭 -> 좌측메뉴 중 'App integration -> App client settings'에 들어가서 상단에 'App client GudiSlsWebApp' 가 나오는 지 확인한다.
    ```

- Cognito 정보를 노출시키기 위한 `Stack Outputs` 추가
  - 외부에서 `userPoolId`, `userPoolClientId`를 가져오게 할 수 있게 `Outputs:`에 Cognito 정보를 넣고 배포한다.

    ```yaml
    # serverless.yml
    resources:
      ...
      Outputs:
        GudiSlsUserPoolId:
          Description: 'Gudi Serverless Framework Handson Cognito User Pool ID'
          Value:
            Ref: 'GudiSlsUserPool'
        GudiSlsUserPoolClientId:
          Description: 'Gudi Serverless Framework Handson Cognito User Pool Client ID'
          Value:
            Ref: 'GudiSlsUserPoolClient'
    ```

    ```sh
    $ npm run deploy

    Stack Outputs
    GudiSlsUserPoolClientId: 752sd3tvmjpbugnt3mh45feis6
    GudiSlsStorageURL: http://gudi-serverless-geoseong.s3-website.ap-northeast-2.amazonaws.com
    GudiSlsUserPoolId: ap-northeast-2_0ZfDbMTHH
    HelloLambdaFunctionQualifiedArn: arn:aws:lambda:ap-northeast-2:623542739657:function:wild-rides-serverless-demo-dev-hello:6
    ServerlessDeploymentBucketName: wild-rides-serverless-de-serverlessdeploymentbuck-1vz2zc0bb43x6
    ```

- 업로드된 웹사이트에서 이용하게 될 backend 정보인 `config.js` 파일 다운로드 후 내용 수정하여 업로드하기
  - 업로드 한 호스팅 페이지 버킷에서 js/config.js 를 다운로드 한 뒤, 자신이 만든 `userPoolId`와 `userPoolClientId`를 채워 넣는다.
    - 로컬환경에 단일 프로필만 존재한다면...  

      ```sh
      # 잘 업로드 되었는지 확인
      $ aws s3 cp s3://gudi-serverless-handson-[내이름]/js/config.js ./
      ```

    - 로컬환경에 다수개의 aws profile이 있고, default가 아닌 다른 profile에 적용하고자 한다면...

      ```sh
      # 잘 업로드 되었는지 확인
      # [AWS프로필명]: OSX(~/.aws/credentials), Windows(`%UserProfile%\.aws/credentials`)
      $ aws s3 cp s3://gudi-serverless-handson-[내이름]/js/config.js ./ --profile [AWS프로필명]
      ```

  ```sh
  aws s3 cp s3://
  ```

  - `config.js`에 `userPoolId`와 `userPoolClientId`를 다 채워 넣었다면, 업로드한 호스팅 버킷 안의 `js/config.js` 경로에 현재 `config.js`을 업로드한다.
    - 로컬환경에 단일 프로필만 존재한다면...  

      ```sh
      # 잘 업로드 되었는지 확인
      $ aws s3 cp ./config.js s3://gudi-serverless-handson-[내이름]/js/config.js
      ```

    - 로컬환경에 다수개의 aws profile이 있고, default가 아닌 다른 profile에 적용하고자 한다면...

      ```sh
      # 잘 업로드 되었는지 확인
      # [AWS프로필명]: OSX(~/.aws/credentials), Windows(`%UserProfile%\.aws/credentials`)
      $ aws s3 cp ./config.js s3://gudi-serverless-handson-[내이름]/js/config.js --profile [AWS프로필명]
      ```

- 회원가입 프로세스 확인하기
  - 이제 호스팅 된 페이지를 들어가서 가장 먼저 보이는 화면의 `Giddy Up!` 버튼을 누르면 `Register 페이지`가 나오는데, 한번 회원가입 프로세스를 진행해 본다.
  
  > 패스워드는 특수문자, 영문 대문자 포함 6글자 이상이여야 한다. 만약 이 policy를 충족 못한다면 alert로 안내문이 알아서 나올 것이다
  
  <img src="https://hands-on.cloud/static/a233232d7a5bd927ad19c1ca2e397ddf/1094d/Serverless-Framework-Cognito-User-Pool-Client-Testing-Registration.png" alt="uploaded s3 static hosting site's register page" />

  - 회원가입할 이메일주소와 패스워드 입력 수 `Let's ryde`버튼을 누르면 입력된 이메일 주소로 인증코드가 날라가며, 페이지는 `ride.html` 로 이동이 되어 있을 것이다.
  <img width="937" alt="001_wildryde_verify" src="https://user-images.githubusercontent.com/19166187/63220571-8ecc9100-c1c5-11e9-8c2d-fb4e1e30d322.png">

  - 인증코드가 정상적으로 입력되면 Signin을 한다.
  <img width="905" alt="002_wildryde_signin" src="https://user-images.githubusercontent.com/19166187/63220572-8f652780-c1c5-11e9-8ef4-0b42c366ef93.png">
  
  - 정상적으로 로그인 된 이후 화면.
  <img width="919" alt="003_wildryde_signedin" src="https://user-images.githubusercontent.com/19166187/63220573-8f652780-c1c5-11e9-8c42-574965110c65.png">

Serverless service backend
- Create AWS DynamoDB table
- Create IAM role for Lambda runction
- Create Lambda function for handling requests
- Validate your implementation

Restful APIs
- Create new REST API
- Create Cognito user pools authorizer
- Create new resource and method
- Create your API deployment
- Global stage declaration
- Updating website config
- Validate your implementation
