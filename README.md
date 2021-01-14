# AWS SAM Lambda Sample app with Cognito

## Introduction

This project contains source code and instructions to create a simple AWS Lambda application as a backend and API Gateway to invoke the Lambda. Also, instructions to configure AWS Cognito to secure the authentication between the API gateway and the client application. Furthermore, We are going to use AWS CodePipeline to configure the CI/CD pipeline. So this will be a fantastic journey. 

## Technologies
- Node.js
- AWS Lambda
- AWS SAM
- AWS Cognito
- AWS API Gateway
- AWS CodeCommit
- AWS CodeBuild
- AWS CodePipeline

## Prerequisites 

1. An AWS Account
2. Text Editor (I am using VS Code because it is incredible.)

## Let us get started

These are the steps that I am going to cover. So, buckle up, folks!!!
1. Create the repository in AWS
2. Configure SAM template(`template.yaml`) file
    1. Configure parameters
    2. Configure Cognito resources
    3. Configure API gateway
    4. Configure the Lambda function
3. Configure `buildspec.yml` file
4. Configure the CI/CD pipeline (AWS CodePipeline)

## 1. Create the repository in AWS

We can use the AWS SAM template to create the project. Please visit [AWS SAM official GitHub](https://github.com/aws/serverless-application-model) page for more details. Otherwise, you can fork this repo and test the functionality.

To see how we can create the AWS CodeCommit repository, please read my [Medium](https://medium.com/@thusithadananjaya) post (Coming Soon).

## 2. Configure SAM template.

Introuduction about SAM

### Configure Parameters

For this example We are going to use these parameters. You can pass these values from the AWS CodePipeline as parameters.

- `IAMExecutionRole` - AWS user role to execute the Lambda.
- `CognitoUserPoolName` - AWS Cognito userpool name.
- `CognitoUserPoolClientName` - The name for AWS Cognito User Pool client.
- `CognitoUserPoolDomain` - The domain prefix that we are going to use to get the token.
- `CognitoUserPoolResourceServerName` - Resource Server name for AWS Cognito User Pool Resource Server.
- `CognitoUserPoolResourceServerIdentifier` - Unique identifier for AWS Cognito User Pool Resource Server.

Configure `template.yaml` file with bellow configs if you need to have a dynamic template.

```yaml
Parameters:
  IAMExecutionRole: 
    Description: IAM role ARN of Lambda execution role.
    Type: String
  CognitoUserPoolName:
    Description: Name for your Cognito user pool
    Type: String
  CognitoUserPoolClientName:
    Description: Name for your Cognito user pool client
    Type: String
  CognitoUserPoolDomain:
    Description: Domain name for your Cognito user pool
    Type: String
  CognitoUserPoolResourceServerName:
    Description: Name for your Cognito user pool Cognito user pool resource server
    Type: String
  CognitoUserPoolResourceServerIdentifier:
    Description: Identifier for your Cognito user pool resource server
    Type: String
```

### Configure Cognito User Pool.

Since we plan to use AWS Cognito to authenticate the client app with the Lambda functions, we need to configure AWS Cognito User Pool.

```yaml
  MyCognitoUserPool: # You can choose your own name.
      Type: AWS::Cognito::UserPool
      Properties:
        UserPoolName: !Ref CognitoUserPoolName # Refferencing User Pool Name parameter.
        Policies:
          PasswordPolicy:
            MinimumLength: 8
        UsernameAttributes:
          - email
        Schema:
          - AttributeDataType: String
            Name: email
            Required: false
```

### Configure Cognito User Client

To get the token first, we need to have a client for our User Pool. You can use the below configurations to create the User Pool Client.

```yaml
  MyCognitoUserPoolClient: # You can choose your own name.
    Type: AWS::Cognito::UserPoolClient
    Properties:
      UserPoolId: !Ref MyCognitoUserPool # Refferencing Cognito User Pool created previously.
      ClientName: !Ref CognitoUserPoolClientName # Refferencing User Pool Client Name parameter.
      GenerateSecret: true
      AllowedOAuthFlows: 
        - client_credentials
      AllowedOAuthFlowsUserPoolClient: true
      AllowedOAuthScopes: 
        - !Join ["/", [!Ref CognitoUserPoolClientName, "create"]] # You can configure Authentication scopes also.
        - !Join ["/", [!Ref CognitoUserPoolClientName, "update"]]
    DependsOn: UserPoolResourceServer
```

### Configure Cognito User Pool Domain

To generate the token, we need to have a valid domain name configure with our AWS Cognito User Pool. You can configure your domain name, or you can choose a domain from AWS.

```yaml
  UserPoolDomain: 
    Type: AWS::Cognito::UserPoolDomain 
    Properties:
      UserPoolId: !Ref MyCognitoUserPool # Refferencing Congnito User Pool domain name parameter. 
      Domain: !Ref CognitoUserPoolDomain
```

### Configure User Pool Resource Server.

Configurations for Cognito user pool resource server.

```yaml
  UserPoolResourceServer: 
    Type: AWS::Cognito::UserPoolResourceServer
    Properties: 
      UserPoolId: !Ref MyCognitoUserPool 
      Identifier:  !Ref CognitoUserPoolResourceServerIdentifier
      Name: !Ref CognitoUserPoolResourceServerName
      Scopes: 
        - ScopeName: "create" 
          ScopeDescription: "create" 
        - ScopeName: "update"
          ScopeDescription: "update"
```

### Configure API Gateway.

Since we will use OAuth 2.0 as the authentication mechanism, wh have to use the `AWS::Serverless::HttpApi` resource type when creating the API gateway.

```yaml
MyApi:
    Type: AWS::Serverless::HttpApi
    Properties:
      StageName: Prod 
      CorsConfiguration: 
        AllowOrigins:
          - "http://*"
          - "https://*"
        AllowHeaders:
          - authorization
        AllowMethods:
          - GET
        MaxAge: 3600
      Auth:
        DefaultAuthorizer: JWTTokenAuthorizer
        Authorizers:
          JWTTokenAuthorizer:
            JwtConfiguration:
              issuer: !Sub https://cognito-idp.${AWS::Region}.amazonaws.com/${MyCognitoUserPool}
              audience: 
                - !Ref MyCognitoUserPoolClient
            IdentitySource: "$request.header.Authorization"
```

### Configure AWS Lambda function.

```yaml
HelloWorldFunction: # You can use any name that you preffered.
    Type: AWS::Serverless::Function 
    Properties:
      CodeUri: hello-world/ # Source code folder.
      Handler: app.lambdaHandler
      Runtime: nodejs12.x
      Role: !Ref IAMExecutionRole # Lambda execution role.
      Events:
        HelloWorld:
          Type: HttpApi
          Properties:
            ApiId: !Ref MyApi # Refferencing API Gateway configurations.
            Path: /hello # API path
            Method: get # HTTP method
```

## 3. Configure `buildspec.yml` file

When you are configuring the AWS CI/CD pipeline, you need to have a repository and instructions to execute when you need to build and store the artifacts. In `buildspec.yaml` you can configure all the build instructions that you need to execute. 

```yaml
version: 0.1
phases:
  install:
    commands:
      - aws cloudformation package --template-file template.yaml --s3-bucket ADD_YOUR_BUCKET_NAME_HERE --output-template-file outputSamTemplate.yaml
artifacts:
  type: zip
  files:
    - template.yaml
    - outputSamTemplate.yaml
```

## 4. Configure CI/CD pipeline (AWS CodePipeline)

Please read my [Medium](https://medium.com/@thusithadananjaya) post(Coming Soon) to configure the CI/CD pipeline. 

 Please add the bellow configuration to the CI/CD Pipeline as overridden parameters.
```json
{
    "IAMExecutionRole": "",
    "CognitoUserPoolName": "",
    "CognitoUserPoolClientName": "",
    "CognitoUserPoolDomain": "",
    "CognitoUserPoolResourceServerName": "",
    "CognitoUserPoolResourceServerIdentifier": ""
}
```
