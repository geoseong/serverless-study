# HandsOn
## Reference
- [Aws Lambda, Amazon Api Gateway, S3, Dynamodb And Cognito Example](https://github.com/andreivmaksimov/serverless-framework-aws-lambda-amazon-api-gateway-s3-dynamodb-and-cognito)
  - [Serverless Framework - Building Web App Using AWS Lambda, Amazon API Gateway S3 DynamoDB And Cognito - Part-1](https://hands-on.cloud/serverless-framework-building-web-app-using-aws-lambda-amazon-api-gateway-s-3-dynamo-db-and-cognito-part-1/)
  - [Serverless Framework - Building Web App Using AWS Lambda, Amazon API Gateway S3 DynamoDB And Cognito - Part-2](https://hands-on.cloud/serverless-framework-building-web-app-using-aws-lambda-amazon-api-gateway-s-3-dynamo-db-and-cognito-part-2/)

## Let's get it
- serverless framework로 프로젝트 생성
  ```
  $ sls create -t aws-nodejs -n wild-rides-serverless-demo
  ```
- npm script를 위하여 npm 프로젝트를 init한다
  ```
  $ npm init -y
  ```
- `serverless.yml`에 **custom 변수**, **S3 Resource** 추가
  ```yaml
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
  ```
  $ sls deploy --aws-profile ["aws프로필명"] --verbose
  or
  $ sls deploy --verbose
  ```

## Upload static content
- 프론트 페이지를 `awslabs/aws-serverless-workshops`로 부터 다운받아 온다
  ```
  $ git clone https://github.com/awslabs/aws-serverless-workshops/
  ```
- 내가 배포했던 S3 Bucket에다가 `WebApplication/1_StaticWebHosting/website` 폴더만 업로드한다
  ```
  $ aws s3 sync ./aws-serverless-workshops/WebApplication/1_StaticWebHosting/website s3://gudi-serverless-handson-[내이름]
  or
  $ aws s3 sync ./aws-serverless-workshops/WebApplication/1_StaticWebHosting/website s3://gudi-serverless-handson-[내이름] --profile [AWS프로필명]
  ```
- 더이상 `aws-serverless-workshops` 폴더는 필요 없으니 지운다
  ```
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
  ```json
  {
    "name": "wild-rides-serverless-demo",
    "version": "1.0.0",
    "description": "",
    "main": "handler.js",
    "scripts": {
      "test": "echo \"Error: no test specified\" && exit 1",
      "deploy": "sls deploy --aws-profile 404hunter-geoseong --verbose",
      "remove": "sls remove --aws-profile 404hunter-geoseong --verbose",
      "package": "sls package"
    },
    "keywords": [],
    "author": "",
    "license": "ISC"
  }
  ``` 
- 미리 작성 해 놓은 **npm 스크립트**를 이용하여 배포하기
  ```
  $ npm run deploy
  ```

## User management