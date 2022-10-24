---
layout: post
title: AWS Landing Zone Accelerator - Integrating Trend Micro Cloud One with AWS Control Tower
---

## Introduction ##

In the past couple of posts we have discussed AWS Landing Zones. I recently had a client engagement where I needed to integrate client's cloud security solution into their AWS Landing Zone. In this short post I will step you through how you integrate Trend Micro Cloud One Workload Security with AWS Control Tower. My client already had Trend Micro Cloud One solution.


## Trend Micro Cloud One ##

Trend Micro Cloud One is available on AWS Marketplace with flexible and easy procurement options and AWS account integration. Under Cloud One there are a number of solutions which are listed below. I'm not going to discuss or provide guidance on the product.

- Conformity
- Workload Security
- Endpoint Security
- Network Security
- Container Security
- Application Security
- File Storage Security
- Open Source Security (snyk)

## Workload Security - What is it? ##

Workload Security consists of the following set of components that work together to provide protection:

- Workload Security console, the centralised web-based management console that administrators use to configure security policy and deploy protection to the enforcement components, the agent.

- The agent is a security agent deployed directly on a computer which provides Application Control, Anti-Malware, Web Reputation service, Firewall, Intrusion Prevention, Integrity Monitoring, and Log Inspection protection to computers on which it is installed.

- The agent contains a Relay module. A relay-enabled agent distributes software and security updates throughout your network of Workload Security components.

- The notifier is an application that communicates information on the local computer about security status and events, and, in the case of relay-enabled agents, also provides information about the security updates being distributed from the local machine.


## How to Integrate Trend Micro Cloud One Workload Security with AWS Control Tower ##

You will want to integrate Workload Security with AWS Control Tower to ensure that every account added through Control Tower Account Factory is automatically provisioned in Workload Security. This will provide centralised visibility to the security posture of EC2 instances deployed in each account.

Using a CloudFormation template which is launched in the Control Tower Master Account, it deploys AWS infrastructure to ensure Workload Security monitors each AWS account automatically. The solution consists of 2 Lambda functions; one to manage our role and access Workload Security, and another to manage the lifecycle of the first Lambda. AWS Secrets Manager is leveraged to store the API key for Workload Security in the Master account and a CloudWatch Events rule is configured to trigger Lambda when a new account is successfully deployed.

1. In the Workload Security console, go to Administration > User Management > API Keys and click New. Select a name for the key and the Full Access role. Be sure to save the key as it cannot be retrieved later. This key will be used to authenticate the automation from the AWS Control Tower Master to the console API.

2. Log into the AWS Control Tower master account. Navigate to the CloudFormation Service, select the region in which AWS Control Tower was deployed, and launch the lifecycle template.

![_config.yml]({{ site.baseurl }}/images/blog/LZ-TrendMicro/BlogImage1.jpg)

##LifeCycle YAML Template##

``` yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Deploy Trend Micro Cloud One Workload Security LifeCycle hook in Control Tower Master v1

Metadata:
  AWS::CloudFormation::Interface:
    ParameterGroups:
      - Label:
          default: "Mandatory Parameters"
        Parameters:
          - ApiKey
      - Label:
          default: "Parameters for Deep Security Manager deployments. Leave defaults for Cloud One Workload Security"
        Parameters:
          - ApiEndpoint
          - DeepSecurityManagerAccount

Parameters:
  ApiKey:
    Type: String
    Description: API key for Cloud One-Workload Security (Using the Workload Security console) or Deep Security manager.
    AllowedPattern : ".+"
    NoEcho: True
  ApiEndpoint:
    Type: String
    Description: "Server FQDN: Port for Deep Security manager. Leave default for Cloud One Workload Security or change for your specific Cloud One region."
    AllowedPattern: ".+"
    Default: 'workload.us-1.cloudone.trendmicro.com'
  DeepSecurityManagerAccount:
    Type: String
    Description: AWS account ID for the shared security accoung where Deep Security Manager is deployed. Leave default for Cloud One Workload Security
    AllowedPattern: '^\d{12}$'
    NoEcho: True
    Default: 147995105371

Mappings:
  LifeCycleRelease:
    Release:
      Stamp: 1593439617

Resources:
  LambdaBucket:
    Type: AWS::S3::Bucket
  CopyLambdaRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: lambda-copier
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action: "s3:GetObject"
                Resource: "arn:aws:s3:::trend-micro-cloud-one-workload-controltower-lifecycle*"
              - Effect: Allow
                Action:
                  - "s3:GetObject"
                  - "s3:PutObject"
                  - "s3:DeleteObject"
                Resource: !Sub 'arn:aws:s3:::${LambdaBucket}*'
              - Effect: "Allow"
                Action:
                  - "cloudformation:DescribeStacks"
                  - "cloudformation:ListStackResources"
                  - "cloudformation:DescribeStackResource"
                Resource: '*'
              - Effect: "Allow"
                Action:
                  - "lambda:UpdateFunctionCode"
                  - "lambda:PublishVersion"
                Resource:
                  - "arn:aws:lambda:*:*:function:*-ConnectorApiLam*"
  CopyLambda:
    Type: Custom::CopyZips
    Properties:
      ServiceToken: !GetAtt 'CopyLambdaFunction.Arn'
      DestBucket: !Ref LambdaBucket
      SourceBucket: trend-micro-cloud-one-workload-controltower-lifecycle
      Nonce: !FindInMap [LifeCycleRelease, Release, Stamp]
      Prefix: ''
      Objects:
        - c1w-controltower-lifecycle.zip
  LifeCycleConfig:
    Type: Custom::CopyResource
    Properties:
      ServiceToken: !GetAtt 'ConnectorApiLambdaFunction.Arn'
      Nonce: !FindInMap [LifeCycleRelease, Release, Stamp]
  CopyLambdaFunction:
    Type: AWS::Lambda::Function
    Properties:
      Description: Copies objects from a source S3 bucket to a destination
      Handler: index.handler
      Runtime: python3.7
      Role: !GetAtt CopyLambdaRole.Arn
      Timeout: 900
      Code:
        ZipFile: |
          import cfnresponse
          import logging
          import boto3
          logger = logging.getLogger(__name__)
          def copy_objects(source_bucket, dest_bucket, prefix, objects):
              s3 = boto3.client('s3')
              for o in objects:
                  key = prefix + o
                  copy_source = {'Bucket': source_bucket, 'Key': key}
                  logging.info(f'copy_source: {copy_source}\ndest_bucket: {dest_bucket}\nkey: {key}')
                  s3.copy_object(CopySource=copy_source, Bucket=dest_bucket, Key=key)
          def delete_objects(bucket, prefix, objects):
              s3 = boto3.client('s3')
              objects = {'Objects': [{'Key': prefix + o} for o in objects]}
              s3.delete_objects(Bucket=bucket, Delete=objects)

          def update_lambda(event, logical_resource_id, object):
              key = event['ResourceProperties']['Prefix'] + event['ResourceProperties']['Objects'][object]
              cloudformationClient = boto3.client('cloudformation')
              lambdaArn = cloudformationClient.describe_stack_resource(StackName=event['StackId'],
                      LogicalResourceId=logical_resource_id)['StackResourceDetail']['PhysicalResourceId']
              logger.info(f'lambdaArn for {logical_resource_id} is {lambdaArn}')
              lambdaClient = boto3.client('lambda')
              response = lambdaClient.update_function_code(FunctionName=lambdaArn,
                      S3Bucket=event['ResourceProperties']['DestBucket'], S3Key=key)
              logger.info(f'Response from update_function_call is: {response}')

          def handler(event, context):
              logger.info(f"Event received by handler: {event}")
              status = cfnresponse.SUCCESS
              try:
                  if event['RequestType'] == 'Delete':
                      delete_objects(event['ResourceProperties']['DestBucket'], event['ResourceProperties']['Prefix'],
                                     event['ResourceProperties']['Objects'])
                  else:
                      copy_objects(event['ResourceProperties']['SourceBucket'], event['ResourceProperties']['DestBucket'],
                                   event['ResourceProperties']['Prefix'], event['ResourceProperties']['Objects'])
                      if event['RequestType'] == 'Update':
                          logger.info(f'RequestType is Update; will update function code')
                          # match logical resource id to item from Custom Resource Objects list
                          update_lambda(event, 'ConnectorApiLambdaFunction', int(0))
              except Exception:
                  logging.error('Unhandled exception', exc_info=True)
                  status = cfnresponse.FAILED
              finally:
                  cfnresponse.send(event, context, status, {}, None)




  TrendMicroCloudOneWorkloadApiLambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - lambda.amazonaws.com
            Action:
              - sts:AssumeRole
      Policies:
        - PolicyName: TMCloudOneWorkloadApiLambdaPolicy
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: "Allow"
                Action:
                  - "logs:CreateLogGroup"
                  - "logs:CreateLogStream"
                  - "logs:PutLogEvents"
                  - "organizations:ListRoots"
                  - "organizations:ListAccounts"
                  - "sts:GetCallerIdentity"
                Resource: "*"
              - Effect: "Allow"
                Action:
                  - "secretsmanager:GetSecretValue"
                  - "secretsmanager:DescribeSecret"
                  - "sts:AssumeRole"
                Resource:
                  - !Ref ApiKeySecret
                  - "arn:aws:iam::*:role/AWSControlTowerExecution"
              - Effect: Allow
                Action: "lambda:InvokeFunction"
                Resource:
                  - "arn:aws:lambda:*:*:function:*-ConnectorApiLam*"
              - Effect: Allow
                Action:
                  - "iam:CreatePolicy"
                  - "iam:GetRole"
                  - "iam:GetPolicyVersion"
                  - "iam:DetachRolePolicy"
                  - "iam:GetPolicy"
                  - "iam:DeletePolicy"
                  - "iam:CreateRole"
                  - "iam:DeleteRole"
                  - "iam:AttachRolePolicy"
                  - "iam:CreatePolicyVersion"
                  - "iam:DeletePolicyVersion"
                  - "iam:SetDefaultPolicyVersion"
                  - "iam:UpdateAssumeRolePolicy"
                Resource:
                  - "arn:aws:iam::*:role/CloudOneWorkloadConnectorRole"
                  - "arn:aws:iam::*:policy/CloudOneWorkloadConnectorPolicy*"
  ConnectorApiLambdaFunction:
    DependsOn: CopyLambda
    Type: AWS::Lambda::Function
    Properties:
      Description: Configures Workload Security Connector for new Control Tower accounts
      Environment:
        Variables:
          ApiEndpoint: !Ref ApiEndpoint
          AccountIdForRole: !Ref DeepSecurityManagerAccount
      Code:
        S3Bucket: !Ref LambdaBucket
        S3Key: c1w-controltower-lifecycle.zip
      Handler: c1w-controltower-lifecycle.lambda_handler
      Runtime: python3.7
      Role: !GetAtt TrendMicroCloudOneWorkloadApiLambdaExecutionRole.Arn
      MemorySize: 128
      Timeout: 120
  ApiKeySecret:
    Type: AWS::SecretsManager::Secret
    Properties:
      Name: 'TrendMicro/CloudOne/WorkloadApiKey'
      SecretString: !Sub |
        {
          "ApiKey": "${ApiKey}"
        }
  EventTriggerLambdaPermission:
    Type: AWS::Lambda::Permission
    Properties:
      Action: lambda:InvokeFunction
      FunctionName: !GetAtt ConnectorApiLambdaFunction.Arn
      Principal: events.amazonaws.com
      SourceArn: !GetAtt EventRuleTrendMicroWorkloadLambdaTrigger.Arn
  EventRuleTrendMicroWorkloadLambdaTrigger:
    DependsOn:
    - ConnectorApiLambdaFunction
    Type: AWS::Events::Rule
    Properties:
      Description: Capture Control Tower LifeCycle Events and Trigger an Action
      EventPattern:
        detail:
          eventName:
          - CreateManagedAccount
          - UpdateManagedAccount
          eventSource:
          - controltower.amazonaws.com
        detail-type:
        - AWS Service Event via CloudTrail
        source:
        - aws.controltower
      Name: C1WorkloadCaptureControlTowerLifeCycleEvents
      State: ENABLED
      Targets:
      - Arn: !GetAtt "ConnectorApiLambdaFunction.Arn"
        Id: IDEventRuleTrendMicroWorkloadLambdaTrigge
```

3. In the lifecycle template, enter your API Key generated in step 1. Leave the FQDN of your console as the default entry.

4. Check the box acknowledging that AWS CloudFormation might create IAM resources. Select Create Stack, and the integration will start adding your AWS accounts to Workload Security.

5. Once all your accounts have been imported, automate agent installation and activate protection within Trend Micro Cloud One.

## Remove AWS Control Tower Integration ##

To remove the lifecycle hook, delete the CloudFormation stack. Protection for Managed Accounts that have already been added will remain in place.

## Trend Micro Cloud One - Future Integrations ##

Trend Micro are currently working on integrating their solution with AWS Security Hub. This will allow you to have all your alerts in a single tool. Please check the Trend Micro documentation for additional information.

## Conclusion ##

This is just a quick summary of how you can integrate Trend Micro Cloud One Workload Security with AWS Control Tower. Once this is completed you will be able to see all AWS accounts in Trend Micro console. The next step is to include the Trend Micro Cloud One Workload Security Agent in your Amazon Machine Images. You can leverage AWS EC2 Image Builder to install the relevant agents in your AMIs. We might cover that in an upcoming post.
