---
layout: post
title: Landing Zone Accelerator on AWS - Part Three Modifying
---

## Landing Zone Accelerator - Introduction ##

Welcome back to the final in our series covering the Landing Zone Accelerator on AWS. In the previous posts I have discussed what the Landing Zone Accelerator is and how to successfully deploy the Landing Zone Accelerator on AWS. In the third part we will discuss how to modify the Landing Zone Accelerator to turn on services and implement your Organisations' cloud policies such as tagging or budget alerts.

## **Warning Costs** ##

You are responsible for the cost of the AWS services used while running this solution. As of August 2022, the cost for running this solution using the Landing Zone Accelerator on AWS best practices configuration with AWS Control Tower in the US East (N. Virginia) Region within a test environment with no active workloads is approximately $357.87 per month.

| AWS Service                                    | Cost per month|
| -------------                                  |:-------------:|
| AWS CloudTrail                                 | $4.30         |
| Amazon CloudWatch                              | $8.56         |
| AWS Config                                     | $24.40        |
| Amazon GuardDuty                               | $4.27         |
| AWS Key Management Service (AWS KMS)           | $110.06       |
| Amazon Macie                                   | $5.50         |
| Amazon Route 53                                | $2.00         |
| Amazon Simple Storage Service (Amazon S3)      | $1.48         |
| Amazon Virtual Private Cloud (Amazon VPC)      | $157.94       |
| AWS Security Hub                               | $38.97        |
| AWS Secrets Manager                            | $0.39         |
| Amazon Simple Notification Service (Amazon SNS)| $0.21         |

## Landing Zone Previous Steps ##

In part two of this series we explained how to deploy the Landing Zone Accelerator. Below are the three steps that need to be completed before moving on to modify the configuration files.

1. The AWSAccelerator-Installer pipeline should show a status of Complete.

2. When the AWSAccelerator-Installer pipeline has completed, a new AWSAccelerator-Pipeline pipeline is created that is now In Progress. Refresh the AWS CodePipeline console if the new pipeline is not visible. The AWSAccelerator-Pipeline pipeline will take approximately 45 minutes to complete. This initial deployment will prepare your environment for Landing Zone Accelerator on AWS and deploy a minimal configuration. Resources deployed include AWS CloudFormation custom resources, CloudWatch Logs log groups for the custom resources, AWS KMS keys for encryption at rest, and Amazon S3 buckets for AWS service logging.

3. After completion of the above steps, your environment is ready to modify.

## Landing Zone Configuration Files ##

In AWS CodeCommit you will see a repository called which contains six files that need to be modified with your required settings. Below are the six files as they will appear after the initial CloudFormation and CodePipeline deployment.

## Global Config ##

```yaml
homeRegion: eu-west-1
enabledRegions:
  - eu-west-1
managementAccountAccessRole: AWSControlTowerExecution
cloudwatchLogRetentionInDays: 3653
centralizeCdkBuckets:
  enable: true
terminationProtection: true
controlTower:
  enable: true
logging:
  account: LogArchive
  cloudtrail:
    enable: false
    organizationTrail: false
    organizationTrailSettings:
      multiRegionTrail: true
      globalServiceEvents: true
      managementEvents: true
      s3DataEvents: true
      lambdaDataEvents: true
      sendToCloudWatchLogs: true
      apiErrorRateInsight: false
      apiCallRateInsight: false
    accountTrails: []
    lifecycleRules: []
  sessionManager:
    sendToCloudWatchLogs: false
    sendToS3: false
    excludeRegions: []
    excludeAccounts: []
    lifecycleRules: []
errors: []
ouIdNames:
  - Root
accountNames: []
```

## Organization Config ##
```yaml
enable: true
organizationalUnits:
  - name: Security
  - name: Infrastructure
serviceControlPolicies: []
taggingPolicies: []
backupPolicies: []
errors: []
```

## Accounts Config ##
```yaml
mandatoryAccounts:
  - name: Management
    description: >-
      The management (primary) account. Do not change the name field for this
      mandatory account. Note, the account name key does not need to match the
      AWS account name.
    email: example@example.com
    organizationalUnit: Root
  - name: LogArchive
    description: >-
      The log archive account. Do not change the name field for this mandatory
      account. Note, the account name key does not need to match the AWS account
      name.
    email: example@example.com
    organizationalUnit: Security
  - name: Audit
    description: >-
      The security audit account (also referred to as the audit account). Do not
      change the name field for this mandatory account. Note, the account name
      key does not need to match the AWS account name.
    email: example@example.com
    organizationalUnit: Security
workloadAccounts: []
errors: []
ouIdNames:
  - Root
```

## IAM Config ##
```yaml
providers: []
policySets: []
roleSets: []
groupSets: []
userSets: []
errors: []
ouIdNames:
  - Root
accountNames: []
```

## Security Config ##
```yaml
centralSecurityServices:
  delegatedAdminAccount: Audit
  ebsDefaultVolumeEncryption:
    enable: false
    excludeRegions: []
  s3PublicAccessBlock:
    enable: false
    excludeAccounts: []
  snsSubscriptions: []
  macie:
    enable: false
    excludeRegions: []
    policyFindingsPublishingFrequency: FIFTEEN_MINUTES
    publishSensitiveDataFindings: true
  guardduty:
    enable: false
    excludeRegions: []
    s3Protection:
      enable: false
      excludeRegions: []
    exportConfiguration:
      enable: false
      destinationType: S3
      exportFrequency: FIFTEEN_MINUTES
  securityHub:
    enable: false
    regionAggregation: false
    excludeRegions: []
    standards: []
  ssmAutomation:
    excludeRegions: []
    documentSets: []
accessAnalyzer:
  enable: false
iamPasswordPolicy:
  allowUsersToChangePassword: true
  hardExpiry: false
  requireUppercaseCharacters: true
  requireLowercaseCharacters: true
  requireSymbols: true
  requireNumbers: true
  minimumPasswordLength: 14
  passwordReusePrevention: 24
  maxPasswordAge: 90
awsConfig:
  enableConfigurationRecorder: true
  enableDeliveryChannel: true
  ruleSets: []
cloudWatch:
  metricSets: []
  alarmSets: []
keyManagementService:
  keySets: []
errors: []
ssmDocuments: []
ouIdNames:
  - Root
accountNames: []
```

## Network Config ##
```yaml
defaultVpc:
  delete: false
  excludeAccounts: []
transitGateways: []
endpointPolicies: []
vpcs: []
vpcFlowLogs:
  trafficType: ALL
  maxAggregationInterval: 600
  destinations:
    - s3
    - cloud-watch-logs
  destinationsConfig:
    s3:
      lifecycleRules: []
    cloudWatchLogs:
      retentionInDays: 3653
  defaultFormat: false
  customFields:
    - version
    - account-id
    - interface-id
    - srcaddr
    - dstaddr
    - srcport
    - dstport
    - protocol
    - packets
    - bytes
    - start
    - end
    - action
    - log-status
    - vpc-id
    - subnet-id
    - instance-id
    - tcp-flags
    - type
    - pkt-srcaddr
    - pkt-dstaddr
    - region
    - az-id
    - pkt-src-aws-service
    - pkt-dst-aws-service
    - flow-direction
    - traffic-path
errors: []
domainLists: []
ouIdNames:
  - Root
accountNames: []
```

## Modifying the YAML Files ##

1. Sign in to the AWS Management Console and navigate to the AWS CodeCommit console. Navigate to the repository named aws-accelerator-configuration. You will be presented with the Landing Zone Accelerator on AWS configuration files.

2. Each configuration file is named based on its purpose in Landing Zone Accelerator on AWS. You need to modify each configuration file to deploy the additional AWS services and infrastructure required. You can use the CodeCommit console or a compatible Git client to manipulate these files. See the appendix for some sample configurations of each file.

3. When finished modifying the configuration files, navigate to the AWS CodePipeline console. Select AWSAccelerator-Pipeline, then Release change. This will kick off a new pipeline instantiation and deploy the configuration changes to your environment.

4. You now need to wait successful completion of the pipeline. If any failures occur, the CodePipeline console displays the failure stage and action in red. To troubleshoot any errors, choose Details on the CodeBuild action to navigate to the failed action. In the CodeBuild console, you can view the Build logs, which will indicate the error encountered during deployment.

## Conclusion ##

At this stage the Landing Zone Accelerator on AWS has been deployed. You have modified the initial deployed configuration with some best practices. This is just the initial building blocks from which you should grow your footprint in AWS. Everything is customisable to meet your requirements. The appendix below has some sample configurations that you can test with. Please see the AWS Solutions page for future updates to this solution. Check back soon for more content and exciting announcement. 

## Appendix Sample Configuration Files ##

## Global Config ##

```yaml
homeRegion: &HOME_REGION us-east-1
enabledRegions:
  - *HOME_REGION
managementAccountAccessRole: AWSControlTowerExecution
cloudwatchLogRetentionInDays: 3653
terminationProtection: true
centralizeCdkBuckets:
  enable: true
controlTower:
  enable: true
logging:
  account: LogArchive
  cloudtrail:
    enable: false
    organizationTrail: false
    organizationTrailSettings:
      multiRegionTrail: true
      globalServiceEvents: true
      managementEvents: true
      s3DataEvents: true
      lambdaDataEvents: true
      sendToCloudWatchLogs: true
      apiErrorRateInsight: false
      apiCallRateInsight: false
    accountTrails:
      - name: AccountTrail
        regions:
          - *HOME_REGION
        deploymentTargets:
          accounts: []
          organizationalUnits:
            - Root
        settings:
          multiRegionTrail: true
          globalServiceEvents: true
          managementEvents: true
          s3DataEvents: true
          lambdaDataEvents: true
          sendToCloudWatchLogs: true
          apiErrorRateInsight: false
          apiCallRateInsight: false
  sessionManager:
    sendToCloudWatchLogs: false
    sendToS3: true
  cloudwatchLogs:
    dynamicPartitioning: dynamic-partitioning/log-filters.json
reports:
  costAndUsageReport:
    compression: Parquet
    format: Parquet
    reportName: accelerator-cur
    s3Prefix: cur
    timeUnit: DAILY
    refreshClosedReports: true
    reportVersioning: CREATE_NEW_REPORT
  budgets:
    - deploymentTargets:
        accounts:
          - Management
      name: accel-budget
      timeUnit: MONTHLY
      type: COST
      amount: 2000
      includeUpfront: true
      includeTax: true
      includeSupport: true
      includeSubscription: true
      includeRecurring: true
      includeOtherSubscription: true
      includeDiscount: true
      includeCredit: false
      includeRefund: false
      useBlended: false
      useAmortized: false
      unit: USD
      notifications:
        - type: ACTUAL
          thresholdType: PERCENTAGE
          threshold: 100
          comparisonOperator: GREATER_THAN
          subscriptionType: EMAIL
          address: <budget-notifications>@example.com  <----- UPDATE EMAIL ADDRESS
        - type: ACTUAL
          thresholdType: PERCENTAGE
          threshold: 90
          comparisonOperator: GREATER_THAN
          subscriptionType: EMAIL
          address: <budget-notifications>@example.com  <----- UPDATE EMAIL ADDRESS
        - type: ACTUAL
          thresholdType: PERCENTAGE
          threshold: 80
          comparisonOperator: GREATER_THAN
          subscriptionType: EMAIL
          address: <budget-notifications>@example.com  <----- UPDATE EMAIL ADDRESS
        - type: ACTUAL
          thresholdType: PERCENTAGE
          threshold: 75
          comparisonOperator: GREATER_THAN
          subscriptionType: EMAIL
          address: <budget-notifications>@example.com  <----- UPDATE EMAIL ADDRESS
        - type: ACTUAL
          thresholdType: PERCENTAGE
          threshold: 50
          comparisonOperator: GREATER_THAN
          subscriptionType: EMAIL
          address: <budget-notifications>@example.com  <----- UPDATE EMAIL ADDRESS
backupVaults:
  - name: BackupVault
    deploymentTargets:
      organizationalUnits:
        - Root
```

## Accounts Config ##

```yaml
mandatoryAccounts:
  # We recommend you do not change mandatory account names. These are used within Landing Zone Accelerator to reference the accounts from other config files.
  # The "name" value does not currently support spaces
  # The "name" value DOES NOT need to match the account name
  - name: Management
    description: The management (primary) account. Do not change the name field for this mandatory account. Note, the account name key does not need to match the AWS account name.
    email: <management-account>@example.com <----- UPDATE EMAIL ADDRESS
    organizationalUnit: Root
  - name: LogArchive
    description: The log archive account. Do not change the name field for this mandatory account. Note, the account name key does not need to match the AWS account name.
    email: <log-archive>@example.com  <----- UPDATE EMAIL ADDRESS
    organizationalUnit: Security
  - name: Audit
    description: The security audit account (also referred to as the audit account). Do not change the name field for this mandatory account. Note, the account name key does not need to match the AWS account name.
    email: <audit>@example.com  <----- UPDATE EMAIL ADDRESS
    organizationalUnit: Security
workloadAccounts:
  # The "name" will be used to set the AWS Account name
  # The "name" value does not currently support spaces
  # The "name" value DOES NOT need to match the account name
  - name: SharedServices
    description: The SharedServices account
    email: <shared-services>@example.com  <----- UPDATE EMAIL ADDRESS
    organizationalUnit: Infrastructure
  - name: Network
    description: The Network account
    email: <network>@example.com  <----- UPDATE EMAIL ADDRESS
    organizationalUnit: Infrastructure
```

## IAM Config ##

```yaml
policySets:
  - deploymentTargets:
      organizationalUnits:
        - Root
    policies:
      - name: Default-Boundary-Policy
        policy: iam-policies/boundary-policy.json
roleSets:
  - deploymentTargets:
      organizationalUnits:
        - Root
    roles:
      - name: EC2-Default-SSM-AD-Role
        instanceProfile: true
        assumedBy:
          - type: service
            principal: ec2.amazonaws.com
        policies:
          awsManaged:
            - AmazonSSMManagedInstanceCore
            - AmazonSSMDirectoryServiceAccess
            - CloudWatchAgentServerPolicy
        boundaryPolicy: Default-Boundary-Policy
      # This role is utilized by the Backup Plans defined in global-config.yaml
      # We create this role in every account where we plan to have Backup Plans
      # and Backup Vaults
      - name: Backup-Role
        assumedBy:
          - type: service
            principal: backup.amazonaws.com
        policies:
          awsManaged:
            - service-role/AWSBackupServiceRolePolicyForBackup
            - service-role/AWSBackupServiceRolePolicyForRestores
groupSets:
  - deploymentTargets:
      organizationalUnits:
        - Root
    groups:
      - name: Administrators
        policies:
          awsManaged:
            - AdministratorAccess
userSets:
  - deploymentTargets:
      accounts:
        - Management
    users:
      - username: breakGlassUser01
        group: Administrators
        boundaryPolicy: Default-Boundary-Policy
      - username: breakGlassUser02
        group: Administrators
        boundaryPolicy: Default-Boundary-Policy
```

## Organisation Config ##

```yaml
# If using AWS Control Tower, ensure that all the specified Organizational Units (OU)
# have been created and enrolled as the accelerator will verify that the OU layout
# matches before continuing to execute the deployment pipeline.

enable: true
organizationalUnits:
  - name: Security
  - name: Infrastructure
quarantineNewAccounts:
  enable: true
  scpPolicyName: Quarantine
serviceControlPolicies:
  - name: AcceleratorGuardrails1
    description: >
      Accelerator GuardRails 1
    policy: service-control-policies/guardrails-1.json
    type: customerManaged
    deploymentTargets:
      organizationalUnits:
        - Infrastructure
  - name: AcceleratorGuardrails2
    description: >
      Accelerator GuardRails 2
    policy: service-control-policies/guardrails-2.json
    type: customerManaged
    deploymentTargets:
      organizationalUnits:
        - Infrastructure
  - name: Quarantine
    description: >
      This SCP is used to prevent changes to new accounts until the Accelerator
      has been executed successfully.
      This policy will be applied upon account creation if enabled.
    policy: service-control-policies/quarantine.json
    type: customerManaged
    deploymentTargets:
      organizationalUnits: []
taggingPolicies:
  - name: TagPolicy
    description: Organization Tagging Policy
    policy: tagging-policies/org-tag-policy.json
    deploymentTargets:
      organizationalUnits:
        - Root
backupPolicies:
  - name: BackupPolicy
    description: Organization Backup Policy
    policy: backup-policies/backup-plan.json
    deploymentTargets:
      organizationalUnits:
        - Root
```

## Security Config ##

```yaml
homeRegion: &HOME_REGION us-east-1
centralSecurityServices:
  delegatedAdminAccount: Audit
  ebsDefaultVolumeEncryption:
    enable: true
    excludeRegions: []
  s3PublicAccessBlock:
    enable: true
    excludeAccounts: []
  snsSubscriptions:
    - level: High
      email: <notify-high>@example.com <----- UPDATE EMAIL ADDRESS
    - level: Medium
      email: <notify-medium>@example.com <----- UPDATE EMAIL ADDRESS
    - level: Low
      email: <notify-low>@example.com <----- UPDATE EMAIL ADDRESS
  macie:
    enable: true
    excludeRegions: []
    policyFindingsPublishingFrequency: FIFTEEN_MINUTES
    publishSensitiveDataFindings: true
  guardduty:
    enable: true
    excludeRegions: []
    s3Protection:
      enable: true
      excludeRegions: []
    exportConfiguration:
      enable: true
      destinationType: S3
      exportFrequency: FIFTEEN_MINUTES
  auditManager:
    enable: true
    excludeRegions: []
    defaultReportsConfiguration:
      enable: true
      destinationType: S3
  detective:
    enable: false
    excludeRegions: []
  securityHub:
    enable: true
    regionAggregation: true
    excludeRegions: []
    standards:
      - name: AWS Foundational Security Best Practices v1.0.0
        enable: true
        controlsToDisable:
          - IAM.1
          - EC2.10
          - Lambda.4
      - name: PCI DSS v3.2.1
        enable: true
        controlsToDisable:
          - PCI.IAM.3
          - PCI.S3.3
          - PCI.EC2.3
          - PCI.Lambda.2
      - name: CIS AWS Foundations Benchmark v1.2.0
        enable: true
        controlsToDisable:
          - CIS.1.20
          - CIS.1.22
          - CIS.2.6
  ssmAutomation:
    documentSets:
      - shareTargets:
          organizationalUnits:
            - Root
        documents:
          # Calls the AWS CLI to enable access logs on a specified ELB
          - name: SSM-ELB-Enable-Logging
            template: ssm-documents/ssm-elb-enable-logging.yaml
          # Enables S3 encryption using KMS
          - name: Put-S3-Encryption
            template: ssm-documents/s3-encryption.yaml
          # Attaches instance profiles to an EC2 instance
          - name: Attach-IAM-Instance-Profile
            template: ssm-documents/attach-iam-instance-profile.yaml
          # Attaches Aws IAM Managed Policy to IAM Role
          - name: Attach-IAM-Role-Policy
            template: ssm-documents/attach-iam-role-policy.yaml
accessAnalyzer:
  enable: true
iamPasswordPolicy:
  allowUsersToChangePassword: true
  hardExpiry: false
  requireUppercaseCharacters: true
  requireLowercaseCharacters: true
  requireSymbols: true
  requireNumbers: true
  minimumPasswordLength: 14
  passwordReusePrevention: 24
  maxPasswordAge: 90
awsConfig:
  enableConfigurationRecorder: true
  enableDeliveryChannel: true
  ruleSets:
    - deploymentTargets:
        organizationalUnits:
          - Root
      rules:
        - name: accelerator-attach-ec2-instance-profile
          type: Custom
          description: Custom rule for checking EC2 instance IAM profile attachment
          inputParameters:
          customRule:
            lambda:
              sourceFilePath: custom-config-rules/attach-ec2-instance-profile.zip
              handler: index.handler
              runtime: nodejs14.x
              rolePolicyFile: custom-config-rules/attach-ec2-instance-profile-detection-role.json
            periodic: true
            maximumExecutionFrequency: Six_Hours
            configurationChanges: true
            triggeringResources:
              lookupType: ResourceTypes
              lookupKey: ResourceTypes
              lookupValue:
                - AWS::EC2::Instance
          remediation:
            rolePolicyFile: custom-config-rules/attach-ec2-instance-profile-remediation-role.json
            automatic: true
            targetId: Attach-IAM-Instance-Profile
            retryAttemptSeconds: 60
            maximumAutomaticAttempts: 5
            parameters:
              - name: InstanceId
                value: RESOURCE_ID
                type: String
              - name: IamInstanceProfile
                value: ${ACCEL_LOOKUP::InstanceProfile:EC2-Default-SSM-AD-Role}
                type: StringList
        - name: accelerator-ec2-instance-profile-permission
          type: Custom
          description: Custom role to remediate EC2 instance profile permission
          inputParameters:
            AWSManagedPolicies: AmazonSSMManagedInstanceCore,AmazonSSMDirectoryServiceAccess,CloudWatchAgentServerPolicy
            #            CustomerManagedPolicies: ${ACCEL_LOOKUP::CustomerManagedPolicy:<POLICY_NAME>},${ACCEL_LOOKUP::CustomerManagedPolicy:<POLICY_NAME>}
            ResourceId: RESOURCE_ID
          customRule:
            lambda:
              sourceFilePath: custom-config-rules/ec2-instance-profile-permissions.zip
              handler: index.handler
              runtime: nodejs14.x
              rolePolicyFile: custom-config-rules/ec2-instance-profile-permissions-detection-role.json
            periodic: true
            maximumExecutionFrequency: Six_Hours
            configurationChanges: true
            triggeringResources:
              lookupType: ResourceTypes
              lookupKey: ResourceTypes
              lookupValue:
                - AWS::IAM::Role
          remediation:
            rolePolicyFile: custom-config-rules/ec2-instance-profile-permissions-remediation-role.json
            automatic: true
            targetId: Attach-IAM-Role-Policy
            targetAccountName: Audit
            retryAttemptSeconds: 60
            maximumAutomaticAttempts: 5
            parameters:
              - name: ResourceId
                value: RESOURCE_ID
                type: String
              - name: AWSManagedPolicies
                value: AmazonSSMManagedInstanceCore,AmazonSSMDirectoryServiceAccess,CloudWatchAgentServerPolicy
                type: StringList
              # - name: CustomerManagedPolicies
              #   value: ${ACCEL_LOOKUP::CustomerManagedPolicy:policy-00},${ACCEL_LOOKUP::CustomerManagedPolicy:policy-01}
              #   type: StringList
        - name: accelerator-s3-bucket-server-side-encryption-enabled
          identifier: S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED
          complianceResourceTypes:
            - AWS::S3::Bucket
          remediation:
            rolePolicyFile: custom-config-rules/bucket-sse-enabled-remediation-role.json
            automatic: true
            targetId: Put-S3-Encryption
            retryAttemptSeconds: 60
            maximumAutomaticAttempts: 5
            parameters:
              - name: BucketName
                value: RESOURCE_ID
                type: String
              - name: KMSMasterKey
                value: ${ACCEL_LOOKUP::KMS}
                type: StringList
        - name: accelerator-elb-logging-enabled
          identifier: ELB_LOGGING_ENABLED
          complianceResourceTypes:
            - AWS::ElasticLoadBalancing::LoadBalancer
            - AWS::ElasticLoadBalancingV2::LoadBalancer
          inputParameters:
            s3BucketNames: ${ACCEL_LOOKUP::Bucket:elbLogs}
          remediation:
            rolePolicyFile: custom-config-rules/elb-logging-enabled-remediation-role.json
            automatic: true
            targetId: SSM-ELB-Enable-Logging
            retryAttemptSeconds: 60
            maximumAutomaticAttempts: 5
            parameters:
              - name: LoadBalancerArn
                value: RESOURCE_ID
                type: String
              - name: LogDestination
                value: ${ACCEL_LOOKUP::Bucket:elbLogs}
                type: StringList
        - name: accelerator-iam-user-group-membership-check
          complianceResourceTypes:
            - AWS::IAM::User
          identifier: IAM_USER_GROUP_MEMBERSHIP_CHECK
        - name: accelerator-securityhub-enabled
          identifier: SECURITYHUB_ENABLED
        - name: accelerator-cloudtrail-enabled
          identifier: CLOUD_TRAIL_ENABLED
        - name: accelerator-rds-logging-enabled
          complianceResourceTypes:
            - AWS::RDS::DBInstance
          identifier: RDS_LOGGING_ENABLED
        - name: accelerator-cloudwatch-alarm-action-check
          complianceResourceTypes:
            - AWS::CloudWatch::Alarm
          inputParameters:
            alarmActionRequired: "TRUE"
            insufficientDataActionRequired: "TRUE"
            okActionRequired: "FALSE"
          identifier: CLOUDWATCH_ALARM_ACTION_CHECK
        - name: accelerator-redshift-cluster-configuration-check
          inputParameters:
            clusterDbEncrypted: "TRUE"
            loggingEnabled: "TRUE"
          complianceResourceTypes:
            - AWS::Redshift::Cluster
          identifier: REDSHIFT_CLUSTER_CONFIGURATION_CHECK
        - name: accelerator-cloudtrail-s3-dataevents-enabled
          identifier: CLOUDTRAIL_S3_DATAEVENTS_ENABLED
        - name: accelerator-emr-kerberos-enabled
          identifier: EMR_KERBEROS_ENABLED
        - name: accelerator-iam-group-has-users-check
          complianceResourceTypes:
            - AWS::IAM::Group
          identifier: IAM_GROUP_HAS_USERS_CHECK
        - name: accelerator-s3-bucket-policy-grantee-check
          complianceResourceTypes:
            - AWS::S3::Bucket
          identifier: S3_BUCKET_POLICY_GRANTEE_CHECK
        - name: accelerator-lambda-inside-vpc
          complianceResourceTypes:
            - AWS::Lambda::Function
          identifier: LAMBDA_INSIDE_VPC
        - name: accelerator-ec2-instances-in-vpc
          complianceResourceTypes:
            - AWS::EC2::Instance
          identifier: INSTANCES_IN_VPC
        - name: accelerator-vpc-sg-open-only-to-authorized-ports
          inputParameters:
            authorizedTcpPorts: "443"
            authorizedUdpPorts: "1020-1025"
          complianceResourceTypes:
            - AWS::EC2::SecurityGroup
          identifier: VPC_SG_OPEN_ONLY_TO_AUTHORIZED_PORTS
        - name: accelerator-ec2-instance-no-public-ip
          complianceResourceTypes:
            - AWS::EC2::Instance
          identifier: EC2_INSTANCE_NO_PUBLIC_IP
        - name: accelerator-elasticsearch-in-vpc-only
          identifier: ELASTICSEARCH_IN_VPC_ONLY
        - name: accelerator-internet-gateway-authorized-vpc-only
          complianceResourceTypes:
            - AWS::EC2::InternetGateway
          identifier: INTERNET_GATEWAY_AUTHORIZED_VPC_ONLY
        - name: accelerator-iam-no-inline-policy-check
          complianceResourceTypes:
            - AWS::IAM::User
            - AWS::IAM::Role
            - AWS::IAM::Group
          identifier: IAM_NO_INLINE_POLICY_CHECK
        - name: accelerator-elb-acm-certificate-required
          complianceResourceTypes:
            - AWS::ElasticLoadBalancing::LoadBalancer
          identifier: ELB_ACM_CERTIFICATE_REQUIRED
        - name: accelerator-alb-http-drop-invalid-header-enabled
          complianceResourceTypes:
            - AWS::ElasticLoadBalancingV2::LoadBalancer
          identifier: ALB_HTTP_DROP_INVALID_HEADER_ENABLED
        - name: accelerator-elb-tls-https-listeners-only
          complianceResourceTypes:
            - AWS::ElasticLoadBalancing::LoadBalancer
          identifier: ELB_TLS_HTTPS_LISTENERS_ONLY
        - name: accelerator-api-gw-execution-logging-enabled
          complianceResourceTypes:
            - AWS::ApiGateway::Stage
            - AWS::ApiGatewayV2::Stage
          identifier: API_GW_EXECUTION_LOGGING_ENABLED
        - name: accelerator-cloudwatch-log-group-encrypted
          identifier: CLOUDWATCH_LOG_GROUP_ENCRYPTED
        - name: accelerator-s3-bucket-replication-enabled
          complianceResourceTypes:
            - AWS::S3::Bucket
          identifier: S3_BUCKET_REPLICATION_ENABLED
        - name: accelerator-cw-loggroup-retention-period-check
          identifier: CW_LOGGROUP_RETENTION_PERIOD_CHECK
        - name: accelerator-ec2-instance-detailed-monitoring-enabled
          complianceResourceTypes:
            - AWS::EC2::Instance
          identifier: EC2_INSTANCE_DETAILED_MONITORING_ENABLED
        - name: accelerator-ec2-volume-inuse-check
          inputParameters:
            deleteOnTermination: "TRUE"
          complianceResourceTypes:
            - AWS::EC2::Volume
          identifier: EC2_VOLUME_INUSE_CHECK
        - name: accelerator-elb-deletion-protection-enabled
          complianceResourceTypes:
            - AWS::ElasticLoadBalancingV2::LoadBalancer
          identifier: ELB_DELETION_PROTECTION_ENABLED
        - name: accelerator-cloudtrail-security-trail-enabled
          identifier: CLOUDTRAIL_SECURITY_TRAIL_ENABLED
        - name: accelerator-elasticache-redis-cluster-automatic-backup-check
          identifier: ELASTICACHE_REDIS_CLUSTER_AUTOMATIC_BACKUP_CHECK
        - name: accelerator-s3-bucket-versioning-enabled
          complianceResourceTypes:
            - AWS::S3::Bucket
          identifier: S3_BUCKET_VERSIONING_ENABLED
        - name: accelerator-vpc-vpn-2-tunnels-up
          complianceResourceTypes:
            - AWS::EC2::VPNConnection
          identifier: VPC_VPN_2_TUNNELS_UP
        - name: accelerator-elb-cross-zone-load-balancing-enabled
          complianceResourceTypes:
            - AWS::ElasticLoadBalancing::LoadBalancer
          identifier: ELB_CROSS_ZONE_LOAD_BALANCING_ENABLED
        - name: accelerator-iam-user-mfa-enabled
          identifier: IAM_USER_MFA_ENABLED
        - name: accelerator-guardduty-non-archived-findings
          inputParameters:
            daysHighSev: "1"
            daysLowSev: "30"
            daysMediumSev: "7"
          identifier: GUARDDUTY_NON_ARCHIVED_FINDINGS
        - name: accelerator-elasticsearch-node-to-node-encryption-check
          complianceResourceTypes:
            - AWS::Elasticsearch::Domain
          identifier: ELASTICSEARCH_NODE_TO_NODE_ENCRYPTION_CHECK
        - name: accelerator-kms-cmk-not-scheduled-for-deletion
          complianceResourceTypes:
            - AWS::KMS::Key
          identifier: KMS_CMK_NOT_SCHEDULED_FOR_DELETION
        - name: accelerator-api-gw-cache-enabled-and-encrypted
          complianceResourceTypes:
            - AWS::ApiGateway::Stage
          identifier: API_GW_CACHE_ENABLED_AND_ENCRYPTED
        - name: accelerator-sagemaker-endpoint-configuration-kms-key-configured
          identifier: SAGEMAKER_ENDPOINT_CONFIGURATION_KMS_KEY_CONFIGURED
        - name: accelerator-sagemaker-notebook-instance-kms-key-configured
          identifier: SAGEMAKER_NOTEBOOK_INSTANCE_KMS_KEY_CONFIGURED
        - name: accelerator-dynamodb-table-encrypted-kms
          complianceResourceTypes:
            - AWS::DynamoDB::Table
          identifier: DYNAMODB_TABLE_ENCRYPTED_KMS
        - name: accelerator-s3-bucket-default-lock-enabled
          complianceResourceTypes:
            - AWS::S3::Bucket
          identifier: S3_BUCKET_DEFAULT_LOCK_ENABLED
cloudWatch:
  metricSets:
    - regions:
        - *HOME_REGION
      #####################################
      # With landing zone version 3.0, AWS Control Tower now supports organization-level AWS CloudTrail trails.                                          #
      # Going forward from landing zone 3.0, AWS Control Tower no longer will support account-level trails that AWS manages.                             #
      # If your environment runs on prior version (3.0) of landing zone, you can change deployment targets for the metrics to Root organizational units  #
      # Metrics deployment target should be management account for environment with landing zone version 3.0                                             #
      # Please refer https://docs.aws.amazon.com/controltower/latest/userguide/2022-all.html#version-3.0 for more information                            #
      #####################################
      deploymentTargets:
        accounts:
          - Management
      metrics:
        # CIS 1.1 – Avoid the use of the "root" account
        - filterName: RootAccountMetricFilter
          logGroupName: aws-controltower/CloudTrailLogs
          filterPattern: '{$.userIdentity.type="Root" && $.userIdentity.invokedBy NOT EXISTS && $.eventType !="AwsServiceEvent"}'
          metricNamespace: LogMetrics
          metricName: RootAccount
          metricValue: "1"
        # CIS 3.1 – Ensure a log metric filter and alarm exist for unauthorized API calls
        - filterName: UnauthorizedAPICallsMetricFilter
          logGroupName: aws-controltower/CloudTrailLogs
          filterPattern: '{($.errorCode="*UnauthorizedOperation") || ($.errorCode="AccessDenied*")}'
          metricNamespace: LogMetrics
          metricName: UnauthorizedAPICalls
          metricValue: "1"
        # CIS 3.2 – Ensure a log metric filter and alarm exist for AWS Management Console sign-in without MFA
        - filterName: ConsoleSigninWithoutMFAMetricFilter
          logGroupName: aws-controltower/CloudTrailLogs
          filterPattern: '{($.eventName = "ConsoleLogin") && ($.additionalEventData.MFAUsed != "Yes") && ($.userIdentity.type = "IAMUser") && ($.responseElements.ConsoleLogin = "Success")}'
          metricNamespace: LogMetrics
          metricName: ConsoleSigninWithoutMFA
          metricValue: "1"
        # CIS 3.3 – Ensure a log metric filter and alarm exist for usage of "root" account
        - filterName: MetricFilter
          logGroupName: aws-controltower/CloudTrailLogs
          filterPattern: '{$.userIdentity.type="Root" && $.userIdentity.invokedBy NOT EXISTS && $.eventType !="AwsServiceEvent"}'
          metricNamespace: LogMetrics
          metricName: RootAccountUsage
          metricValue: "1"
        # CIS 3.4 – Ensure a log metric filter and alarm exist for IAM policy changes
        - filterName: IAMPolicyChangesMetricFilter
          logGroupName: aws-controltower/CloudTrailLogs
          filterPattern: "{($.eventName=DeleteGroupPolicy) || ($.eventName=DeleteRolePolicy) || ($.eventName=DeleteUserPolicy) || ($.eventName=PutGroupPolicy) || ($.eventName=PutRolePolicy) || ($.eventName=PutUserPolicy) || ($.eventName=CreatePolicy) || ($.eventName=DeletePolicy) || ($.eventName=CreatePolicyVersion) || ($.eventName=DeletePolicyVersion) || ($.eventName=AttachRolePolicy) || ($.eventName=DetachRolePolicy) || ($.eventName=AttachUserPolicy) || ($.eventName=DetachUserPolicy) || ($.eventName=AttachGroupPolicy) || ($.eventName=DetachGroupPolicy)}"
          metricNamespace: LogMetrics
          metricName: IAMPolicyChanges
          metricValue: "1"
        # CIS 3.5 – Ensure a log metric filter and alarm exist for CloudTrail configuration changes
        - filterName: CloudTrailChangesMetricFilter
          logGroupName: aws-controltower/CloudTrailLogs
          filterPattern: "{($.eventName=CreateTrail) || ($.eventName=UpdateTrail) || ($.eventName=DeleteTrail) || ($.eventName=StartLogging) || ($.eventName=StopLogging)}"
          metricNamespace: LogMetrics
          metricName: CloudTrailChanges
          metricValue: "1"
        # CIS 3.6 – Ensure a log metric filter and alarm exist for AWS Management Console authentication failures
        - filterName: ConsoleAuthenticationFailureMetricFilter
          logGroupName: aws-controltower/CloudTrailLogs
          filterPattern: '{($.eventName=ConsoleLogin) && ($.errorMessage="Failed authentication")}'
          metricNamespace: LogMetrics
          metricName: ConsoleAuthenticationFailure
          metricValue: "1"
        # CIS 3.7 – Ensure a log metric filter and alarm exist for disabling or scheduled deletion of customer created CMKs
        - filterName: DisableOrDeleteCMKMetricFilter
          logGroupName: aws-controltower/CloudTrailLogs
          filterPattern: "{($.eventSource=kms.amazonaws.com) && (($.eventName=DisableKey) || ($.eventName=ScheduleKeyDeletion))}"
          metricNamespace: LogMetrics
          metricName: DisableOrDeleteCMK
          metricValue: "1"
        # CIS 3.8 – Ensure a log metric filter and alarm exist for S3 bucket policy changes
        - filterName: S3BucketPolicyChangesMetricFilter
          logGroupName: aws-controltower/CloudTrailLogs
          filterPattern: "{($.eventSource=s3.amazonaws.com) && (($.eventName=PutBucketAcl) || ($.eventName=PutBucketPolicy) || ($.eventName=PutBucketCors) || ($.eventName=PutBucketLifecycle) || ($.eventName=PutBucketReplication) || ($.eventName=DeleteBucketPolicy) || ($.eventName=DeleteBucketCors) || ($.eventName=DeleteBucketLifecycle) || ($.eventName=DeleteBucketReplication))}"
          metricNamespace: LogMetrics
          metricName: S3BucketPolicyChanges
          metricValue: "1"
        # CIS 3.9 – Ensure a log metric filter and alarm exist for AWS Config configuration changes
        - filterName: AWSConfigChangesMetricFilter
          logGroupName: aws-controltower/CloudTrailLogs
          filterPattern: "{($.eventSource=config.amazonaws.com) && (($.eventName=StopConfigurationRecorder) || ($.eventName=DeleteDeliveryChannel) || ($.eventName=PutDeliveryChannel) || ($.eventName=PutConfigurationRecorder))}"
          metricNamespace: LogMetrics
          metricName: AWSConfigChanges
          metricValue: "1"
        # CIS 3.10 – Ensure a log metric filter and alarm exist for security group changes
        - filterName: SecurityGroupChangesMetricFilter
          logGroupName: aws-controltower/CloudTrailLogs
          filterPattern: "{($.eventName=AuthorizeSecurityGroupIngress) || ($.eventName=AuthorizeSecurityGroupEgress) || ($.eventName=RevokeSecurityGroupIngress) || ($.eventName=RevokeSecurityGroupEgress) || ($.eventName=CreateSecurityGroup) || ($.eventName=DeleteSecurityGroup)}"
          metricNamespace: LogMetrics
          metricName: SecurityGroupChanges
          metricValue: "1"
        # CIS 3.11 – Ensure a log metric filter and alarm exist for changes to Network Access Control Lists (NACL)
        - filterName: NetworkACLChangesMetricFilter
          logGroupName: aws-controltower/CloudTrailLogs
          filterPattern: "{($.eventName=CreateNetworkAcl) || ($.eventName=CreateNetworkAclEntry) || ($.eventName=DeleteNetworkAcl) || ($.eventName=DeleteNetworkAclEntry) || ($.eventName=ReplaceNetworkAclEntry) || ($.eventName=ReplaceNetworkAclAssociation)}"
          metricNamespace: LogMetrics
          metricName: NetworkACLChanges
          metricValue: "1"
        # CIS 3.12 – Ensure a log metric filter and alarm exist for changes to network gateways
        - filterName: NetworkGatewayChangesMetricFilter
          logGroupName: aws-controltower/CloudTrailLogs
          filterPattern: "{($.eventName=CreateCustomerGateway) || ($.eventName=DeleteCustomerGateway) || ($.eventName=AttachInternetGateway) || ($.eventName=CreateInternetGateway) || ($.eventName=DeleteInternetGateway) || ($.eventName=DetachInternetGateway)}"
          metricNamespace: LogMetrics
          metricName: NetworkGatewayChanges
          metricValue: "1"
        # CIS 3.13 – Ensure a log metric filter and alarm exist for route table changes
        - filterName: RouteTableChangesMetricFilter
          logGroupName: aws-controltower/CloudTrailLogs
          filterPattern: "{($.eventName=CreateRoute) || ($.eventName=CreateRouteTable) || ($.eventName=ReplaceRoute) || ($.eventName=ReplaceRouteTableAssociation) || ($.eventName=DeleteRouteTable) || ($.eventName=DeleteRoute) || ($.eventName=DisassociateRouteTable)}"
          metricNamespace: LogMetrics
          metricName: RouteTableChanges
          metricValue: "1"
        # CIS 3.14 – Ensure a log metric filter and alarm exist for VPC changes
        - filterName: VPCChangesMetricFilter
          logGroupName: aws-controltower/CloudTrailLogs
          filterPattern: "{($.eventName=CreateVpc) || ($.eventName=DeleteVpc) || ($.eventName=ModifyVpcAttribute) || ($.eventName=AcceptVpcPeeringConnection) || ($.eventName=CreateVpcPeeringConnection) || ($.eventName=DeleteVpcPeeringConnection) || ($.eventName=RejectVpcPeeringConnection) || ($.eventName=AttachClassicLinkVpc) || ($.eventName=DetachClassicLinkVpc) || ($.eventName=DisableVpcClassicLink) || ($.eventName=EnableVpcClassicLink)}"
          metricNamespace: LogMetrics
          metricName: VPCChanges
          metricValue: "1"
  alarmSets:
    - regions:
        - *HOME_REGION
      #####################################
      # With landing zone version 3.0, AWS Control Tower now supports organization-level AWS CloudTrail trails.                                          #
      # Going forward from landing zone 3.0, AWS Control Tower no longer will support account-level trails that AWS manages.                             #
      # If your environment runs on prior version (3.0) of landing zone, you can change deployment targets for the metrics to Root organizational units  #
      # Metrics deployment target should be management account for environment with landing zone version 3.0                                             #
      # Please refer https://docs.aws.amazon.com/controltower/latest/userguide/2022-all.html#version-3.0 for more information                            #
      #####################################
      deploymentTargets:
        accounts:
          - Management
      alarms:
        # CIS 1.1 – Avoid the use of the "root" account
        - alarmName: CIS-1.1-RootAccountUsage
          alarmDescription: Alarm for usage of "root" account
          snsAlertLevel: Low
          metricName: RootAccountUsage
          namespace: LogMetrics
          comparisonOperator: GreaterThanOrEqualToThreshold
          evaluationPeriods: 1
          period: 300
          statistic: Sum
          threshold: 1
          treatMissingData: notBreaching
        # CIS 3.1 – Ensure a log metric filter and alarm exist for unauthorized API calls
        - alarmName: CIS-3.1-UnauthorizedAPICalls
          alarmDescription: Alarm for unauthorized API calls
          snsAlertLevel: Low
          metricName: UnauthorizedAPICalls
          namespace: LogMetrics
          comparisonOperator: GreaterThanOrEqualToThreshold
          evaluationPeriods: 1
          period: 300
          statistic: Average
          threshold: 5
          treatMissingData: notBreaching
        # CIS 3.2 – Ensure a log metric filter and alarm exist for AWS Management Console sign-in without MFA
        - alarmName: CIS-3.2-ConsoleSigninWithoutMFA
          alarmDescription: Alarm for AWS Management Console sign-in without MFA
          snsAlertLevel: Low
          metricName: ConsoleSigninWithoutMFA
          namespace: LogMetrics
          comparisonOperator: GreaterThanOrEqualToThreshold
          evaluationPeriods: 1
          period: 300
          statistic: Sum
          threshold: 1
          treatMissingData: notBreaching
        # CIS 3.3 – Ensure a log metric filter and alarm exist for usage of "root" account
        - alarmName: CIS-3.3-RootAccountUsage
          alarmDescription: Alarm for usage of "root" account
          snsAlertLevel: Low
          metricName: RootAccountUsage
          namespace: LogMetrics
          comparisonOperator: GreaterThanOrEqualToThreshold
          evaluationPeriods: 1
          period: 300
          statistic: Sum
          threshold: 1
          treatMissingData: notBreaching
        # CIS 3.4 – Ensure a log metric filter and alarm exist for IAM policy changes
        - alarmName: CIS-3.4-IAMPolicyChanges
          alarmDescription: Alarm for IAM policy changes
          snsAlertLevel: Low
          metricName: IAMPolicyChanges
          namespace: LogMetrics
          comparisonOperator: GreaterThanOrEqualToThreshold
          evaluationPeriods: 1
          period: 300
          statistic: Average
          threshold: 1
          treatMissingData: notBreaching
        # CIS 3.5 – Ensure a log metric filter and alarm exist for CloudTrail configuration changes
        - alarmName: CIS-3.5-CloudTrailChanges
          alarmDescription: Alarm for CloudTrail configuration changes
          snsAlertLevel: Low
          metricName: CloudTrailChanges
          namespace: LogMetrics
          comparisonOperator: GreaterThanOrEqualToThreshold
          evaluationPeriods: 1
          period: 300
          statistic: Sum
          threshold: 1
          treatMissingData: notBreaching
        # CIS 3.6 – Ensure a log metric filter and alarm exist for AWS Management Console authentication failures
        - alarmName: CIS-3.6-ConsoleAuthenticationFailure
          alarmDescription: Alarm exist for AWS Management Console authentication failures
          snsAlertLevel: Low
          metricName: ConsoleAuthenticationFailure
          namespace: LogMetrics
          comparisonOperator: GreaterThanOrEqualToThreshold
          evaluationPeriods: 1
          period: 300
          statistic: Sum
          threshold: 1
          treatMissingData: notBreaching
        # CIS 3.7 – Ensure a log metric filter and alarm exist for disabling or scheduled deletion of customer created CMKs
        - alarmName: CIS-3.7-DisableOrDeleteCMK
          alarmDescription: Alarm for disabling or scheduled deletion of customer created CMKs
          snsAlertLevel: Low
          metricName: DisableOrDeleteCMK
          namespace: LogMetrics
          comparisonOperator: GreaterThanOrEqualToThreshold
          evaluationPeriods: 1
          period: 300
          statistic: Sum
          threshold: 1
          treatMissingData: notBreaching
        # CIS 3.8 – Ensure a log metric filter and alarm exist for S3 bucket policy changes
        - alarmName: CIS-3.8-S3BucketPolicyChanges.
          alarmDescription: Alarm for S3 bucket policy changes
          snsAlertLevel: Low
          metricName: S3BucketPolicyChanges
          namespace: LogMetrics
          comparisonOperator: GreaterThanOrEqualToThreshold
          evaluationPeriods: 1
          period: 300
          statistic: Average
          threshold: 1
          treatMissingData: notBreaching
        # CIS 3.9 – Ensure a log metric filter and alarm exist for AWS Config configuration changes
        - alarmName: CIS-3.9-AWSConfigChanges
          alarmDescription: Alarm for AWS Config configuration changes
          snsAlertLevel: Low
          metricName: AWSConfigChanges
          namespace: LogMetrics
          comparisonOperator: GreaterThanOrEqualToThreshold
          evaluationPeriods: 1
          period: 300
          statistic: Sum
          threshold: 1
          treatMissingData: notBreaching
        # CIS 3.10 – Ensure a log metric filter and alarm exist for security group changes
        - alarmName: CIS-3.10-SecurityGroupChanges
          alarmDescription: Alarm for security group changes
          snsAlertLevel: Low
          metricName: SecurityGroupChanges
          namespace: LogMetrics
          comparisonOperator: GreaterThanOrEqualToThreshold
          evaluationPeriods: 1
          period: 300
          statistic: Sum
          threshold: 1
          treatMissingData: notBreaching
        # CIS 3.11 – Ensure a log metric filter and alarm exist for changes to Network Access Control Lists (NACL)
        - alarmName: CIS-3.11-NetworkACLChanges
          alarmDescription: Alarm for changes to Network Access Control Lists (NACL)
          snsAlertLevel: Low
          metricName: NetworkACLChanges
          namespace: LogMetrics
          comparisonOperator: GreaterThanOrEqualToThreshold
          evaluationPeriods: 1
          period: 300
          statistic: Sum
          threshold: 1
          treatMissingData: notBreaching
        # CIS 3.12 – Ensure a log metric filter and alarm exist for changes to network gateways
        - alarmName: CIS-3.12-NetworkGatewayChanges
          alarmDescription: Alarm for changes to network gateways
          snsAlertLevel: Low
          metricName: NetworkGatewayChanges
          namespace: LogMetrics
          comparisonOperator: GreaterThanOrEqualToThreshold
          evaluationPeriods: 1
          period: 300
          statistic: Sum
          threshold: 1
          treatMissingData: notBreaching
        # CIS 3.13 – Ensure a log metric filter and alarm exist for route table changes
        - alarmName: CIS-3.13-RouteTableChanges
          alarmDescription: Alarm for route table changes
          snsAlertLevel: Low
          metricName: RouteTableChanges
          namespace: LogMetrics
          comparisonOperator: GreaterThanOrEqualToThreshold
          evaluationPeriods: 1
          period: 300
          statistic: Average
          threshold: 1
          treatMissingData: notBreaching
        # CIS 3.14 – Ensure a log metric filter and alarm exist for VPC changes
        - alarmName: CIS-3.14-VPCChanges
          alarmDescription: Alarm for VPC changes
          snsAlertLevel: Low
          metricName: VPCChanges
          namespace: LogMetrics
          comparisonOperator: GreaterThanOrEqualToThreshold
          evaluationPeriods: 1
          period: 300
          statistic: Sum
          threshold: 1
          treatMissingData: notBreaching
```

## Network Config ##

```yaml
homeRegion: &HOME_REGION us-east-1
#####################################
# Delete default VPCs-- use this    #
# object to delete default VPCs in  #
# any non-excluded accounts         #
#####################################
defaultVpc:
  delete: true
  excludeAccounts: []

#####################################
# Transit Gateways-- use this object #
# to deploy transit gateways         #
#####################################
transitGateways:
  - name: Network-Main
    account: Network
    region: *HOME_REGION
    shareTargets:
      organizationalUnits:
        - Infrastructure
    asn: 65521
    dnsSupport: enable
    vpnEcmpSupport: enable
    defaultRouteTableAssociation: disable
    defaultRouteTablePropagation: disable
    autoAcceptSharingAttachments: enable
    routeTables:
      - name: Network-Main-Core
        routes: []
        # Use routes to define static routes for TGW route tables.
        # - destinationCidrBlock: 10.100.0.0/16
        #     attachment:
        #       vpcName: Network-Endpoints
        #       account: Network
        # - destinationCidrBlock: 10.200.0.0/16
        #     attachment:
        #       directConnectGatewayName: Network-DXGW
        # - destinationCidrBlock: 1.1.1.1/32
        #     blackhole: true
        # - destinationPrefixList: accelerator-prefix-list
        #   attachment:
        #     vpcName: Network-Endpoints
        #     account: Network
      - name: Network-Main-Segregated
        routes: []
      - name: Network-Main-Shared
        routes: []
      - name: Network-Main-Standalone
        routes: []
#####################################
# DHCP options -- use this object   #
# to define DHCP options sets to use #
# for VPCs.                         #
#####################################
# dhcpOptions:
#   - name: accelerator-dhcp-opts
#     accounts:
#       - Network
#     regions:
#       - *HOME_REGION
#     domainName: example.com
#     domainNameServers:
#       - 1.1.1.1
#       - 2.2.2.2
#     netbiosNameServers:
#       - 1.1.1.1
#       - 2.2.2.2
#     netbiosNodeType: 2
#     ntpServers:
#       - 1.1.1.1
#       - 2.2.2.2

#####################################
# Central network services -- use   #
# this object to define a delegated  #
# admin account and configure        #
# advanced networking services      #
#####################################
# centralNetworkServices:
#   delegatedAdminAccount: Network
#   ipams:
#     - name: accelerator-ipam
#       region: *HOME_REGION
#       description: Accelerator IPAM
#       operatingRegions:
#         - *HOME_REGION
#         - us-west-2
#       pools:
#         - name: &BASE_POOL base-pool
#           description: accelerator-base
#           provisionedCidrs:
#             - 10.0.0.0/8
#             - 172.16.0.0/12
#             - 192.168.0.0/16
#         - name: home-region-pool
#           description: Pool for us-east-1
#           locale: *HOME_REGION
#           provisionedCidrs:
#             - 10.0.0.0/16
#           sourceIpamPool: *BASE_POOL
#         - name: home-region-prod-pool
#           description: Pool for prod environment
#           allocationResourceTags:
#             - key: env
#               value: prod
#           locale: *HOME_REGION
#           provisionedCidrs:
#             - 10.0.0.0/24
#           shareTargets:
#             organizationalUnits:
#               - Infrastructure
#           sourceIpamPool: home-region-pool
#         - name: west-region-pool
#           description: Pool for us-west-2
#           locale: us-west-2
#           provisionedCidrs:
#             - 10.1.0.0/16
#           sourceIpamPool: *BASE_POOL
#
#   networkFirewall:
#     firewalls:
#       - name: accelerator-firewall
#         region: *HOME_REGION
#         firewallPolicy: accelerator-policy
#         subnets:
#           - Network-Inspection-Firewall-A
#           - Network-Inspection-Firewall-B
#         vpc: Network-Inspection
#         loggingConfiguration:
#           - destination: s3
#             type: ALERT
#           - destination: cloud-watch-logs
#             type: FLOW
#     policies:
#       - name: accelerator-policy
#         regions:
#           - *HOME_REGION
#         firewallPolicy:
#           statelessDefaultActions: ['aws:forward_to_sfe']
#           statelessFragmentDefaultActions: ['aws:forward_to_sfe']
#           statefulRuleGroups:
#             - name: accelerator-rule-group
#             - name: domain-list-group
#         shareTargets:
#           organizationalUnits:
#             - Infrastructure
#     rules:
#       - name: accelerator-rule-group
#         regions:
#           - *HOME_REGION
#         capacity: 100
#         type: STATEFUL
#         ruleGroup:
#           rulesSource:
#             statefulRules:
#               - action: PASS
#                 header:
#                   destination: 10.0.0.0/16
#                   destinationPort: ANY
#                   direction: FORWARD
#                   protocol: IP
#                   source: 10.1.0.0/16
#                   sourcePort: ANY
#                 ruleOptions:
#                   - keyword: sid
#                     settings: ['100']
#               - action: DROP
#                 header:
#                   destination: ANY
#                   destinationPort: ANY
#                   direction: ANY
#                   protocol: IP
#                   source: ANY
#                   sourcePort: ANY
#                 ruleOptions:
#                   - keyword: sid
#                     settings: ['101']
#               - action: ALERT
#                 header:
#                   destination: 1.1.1.1/32
#                   destinationPort: '80'
#                   direction: FORWARD
#                   protocol: TCP
#                   source: ANY
#                   sourcePort: ANY
#                 ruleOptions:
#                   - keyword: msg
#                     settings: ['"example message"']
#                   - keyword: sid
#                     settings: ['102']
#       - name: domain-list-group
#         regions:
#           - *HOME_REGION
#         capacity: 10
#         type: STATEFUL
#         ruleGroup:
#           rulesSource:
#             rulesSourceList:
#               generatedRulesType: DENYLIST
#               targets: ['.example.com']
#               targetTypes: ['TLS_SNI', 'HTTP_HOST']
#           ruleVariables:
#             ipSets:
#               name: HOME_NET
#               definition:
#                 - 10.0.0.0/16
#                 - 10.1.0.0/16
#             portSets:
#               name: HOME_NET
#               definition:
#                 - '80'
#                 - '443'
#
#   gatewayLoadBalancers:
#     - name: Accelerator-GWLB
#       subnets:
#         - Network-Inspection-A
#         - Network-Inspection-B
#       vpc: Network-Inspection
#       deletionProtection: true
#       endpoints:
#         - name: Endpoint-A
#           account: Network
#           subnet: Network-Inspection-A
#           vpc: Network-Inspection
#         - name: Endpoint-B
#           account: Network
#           subnet: Network-Inspection-B
#           vpc: Network-Inspection
#
#   route53Resolver:
#     endpoints:
#       - name: accelerator-inbound
#         type: INBOUND
#         vpc: Network-Endpoints
#         subnets:
#           - Network-Endpoints-A
#           - Network-Endpoints-B
#       - name: accelerator-outbound
#         type: OUTBOUND
#         vpc: Network-Endpoints
#         subnets:
#           - Network-Endpoints-A
#           - Network-Endpoints-B
#         rules:
#           - name: example-rule
#             domainName: example.com
#             targetIps:
#               - ip: 1.1.1.1
#                 port: '5353' # only include if targeting a non-standard DNS port
#               - ip: 2.2.2.2
#             shareTargets:
#               organizationalUnits:
#                 - Infrastructure
#           - name: inbound-target-rule
#             domainName: aws.internal.domain
#             inboundEndpointTarget: accelerator-inbound # This endpoint must be listed in the configuration before the outbound endpoint
#     queryLogs:
#       name: accelerator-query-logs
#       destinations:
#         - s3
#         - cloud-watch-logs
#       shareTargets:
#         organizationalUnits:
#           - Infrastructure
#     firewallRuleGroups:
#       - name: accelerator-block-group
#         regions:
#           - *HOME_REGION
#         rules:
#           - name: nxdomain-block-rule
#             action: BLOCK
#             customDomainList: dns-firewall-domain-lists/domain-list-1.txt
#             priority: 100
#             blockResponse: NXDOMAIN
#           - name: override-block-rule
#             action: BLOCK
#             customDomainList: dns-firewall-domain-lists/domain-list-2.txt
#             priority: 200
#             blockResponse: OVERRIDE
#             blockOverrideDomain: amazon.com
#             blockOverrideTtl: 3600
#           - name: managed-rule
#             action: BLOCK
#             managedDomainList: AWSManagedDomainsBotnetCommandandControl
#             priority: 300
#             blockResponse: NODATA
#         shareTargets:
#           organizationalUnits:
#             - Infrastructure

#####################################
# Prefix lists -- use this object to #
# deploy prefix lists to be used for #
# security groups, subnet routes,   #
# and/or TGW static routes          #
#####################################
# prefixLists:
#   - name: accelerator-prefix-list
#     accounts:
#       - Network
#     regions:
#       - *HOME_REGION
#     addressFamily: 'IPv4'
#     maxEntries: 1
#     entries:
#       - 10.1.0.1/32

#####################################
# Endpoint policies -- use this     #
# object to define standard policies #
# for VPC endpoints                 #
#####################################
endpointPolicies:
  - name: Default
    document: vpc-endpoint-policies/default.json
  - name: Ec2
    document: vpc-endpoint-policies/ec2.json

#####################################
# VPC templates -- use this object  #
# to deploy standard VPCs across    #
# multiple accounts/OUs             #
#####################################
# vpcTemplates:
#   - name: Accelerator-Template
#     region: *HOME_REGION
#     deploymentTargets:
#       organizationalUnits:
#         - Infrastructure
#     ipamAllocations:
#       - ipamPoolName: home-region-prod-pool
#         netmaskLength: 25
#     internetGateway: false
#     enableDnsHostnames: true
#     enableDnsSupport: true
#     instanceTenancy: default
#     routeTables:
#       - name: Accelerator-Template-Tgw-A
#         routes: []
#       - name: Accelerator-Template-Tgw-B
#         routes: []
#       - name: Accelerator-Template-A
#         routes:
#           - name: TgwRoute
#             destination: 0.0.0.0/0
#             type: transitGateway
#             target: Network-Main
#       - name: Accelerator-Template-B
#         routes:
#           - name: TgwRoute
#             destination: 0.0.0.0/0
#             type: transitGateway
#             target: Network-Main
#     subnets:
#       - name: Accelerator-Template-A
#         availabilityZone: a
#         routeTable: Accelerator-Template-A
#         ipamAllocation:
#           ipamPoolName: home-region-prod-pool
#           netmaskLength: 27
#       - name: Accelerator-Template-B
#         availabilityZone: b
#         routeTable: Accelerator-Template-B
#         ipamAllocation:
#           ipamPoolName: home-region-prod-pool
#           netmaskLength: 27
#       - name: Accelerator-TemplateTgwAttach-A
#         availabilityZone: a
#         routeTable: Accelerator-Template-Tgw-A
#         ipamAllocation:
#           ipamPoolName: home-region-prod-pool
#           netmaskLength: 27
#       - name: Accelerator-TemplateTgwAttach-B
#         availabilityZone: b
#         routeTable: Accelerator-Template-Tgw-B
#         ipamAllocation:
#           ipamPoolName: home-region-prod-pool
#           netmaskLength: 27
#     transitGatewayAttachments:
#       - name: Accelerator-Template
#         transitGateway:
#           name: Network-Main
#           account: Network
#         routeTableAssociations:
#           - Network-Main-Shared
#         routeTablePropagations:
#           - Network-Main-Core
#           - Network-Main-Shared
#           - Network-Main-Segregated
#         subnets:
#           - Accelerator-TemplateTgwAttach-A
#           - Accelerator-TemplateTgwAttach-B
#     tags:
#       - key: env
#         value: prod

#####################################
# VPCs-- use this object to deploy  #
# a VPC in a single account and     #
# region.                           #
#####################################
vpcs:
  - name: Network-Endpoints
    account: Network
    region: *HOME_REGION
    cidrs:
      - 10.1.0.0/22
    # ipamAllocations:
    #   - ipamPoolName: home-region-prod-pool
    #     netmaskLength: 25
    internetGateway: false
    # dhcpOptions: accelerator-dhcp-opts
    enableDnsHostnames: true
    enableDnsSupport: true
    instanceTenancy: default
    # dnsFirewallRuleGroups:
    #   - name: accelerator-block-group
    #     priority: 101
    # queryLogs:
    #   - accelerator-query-logs
    # resolverRules:
    #   - example-rule
    routeTables:
      - name: Network-Endpoints-Tgw-A
        routes: []
      - name: Network-Endpoints-Tgw-B
        routes: []
      - name: Network-Endpoints-A
        routes:
          - name: TgwRoute
            destination: 0.0.0.0/0
            type: transitGateway
            target: Network-Main
          - name: S3Gateway
            type: gatewayEndpoint
            target: s3
          - name: DynamoDBGateway
            type: gatewayEndpoint
            target: dynamodb
          # - name: PlRoute
          #   destinationPrefixList: accelerator-prefix-list
          #   type: transitGateway
          #   target: Network-Main
      - name: Network-Endpoints-B
        routes:
          - name: TgwRoute
            destination: 0.0.0.0/0
            type: transitGateway
            target: Network-Main
          - name: S3Gateway
            type: gatewayEndpoint
            target: s3
          - name: DynamoDBGateway
            type: gatewayEndpoint
            target: dynamodb
          # - name: PlRoute
          #   destinationPrefixList: accelerator-prefix-list
          #   type: transitGateway
          #   target: Network-Main
    subnets:
      - name: Network-Endpoints-A
        availabilityZone: a
        routeTable: Network-Endpoints-A
        ipv4CidrBlock: 10.1.0.0/24
        # ipamAllocation:
        #   ipamPoolName: home-region-prod-pool
        #   netmaskLength: 27
      - name: Network-Endpoints-B
        availabilityZone: b
        routeTable: Network-Endpoints-B
        ipv4CidrBlock: 10.1.1.0/24
      - name: Network-EndpointsTgwAttach-A
        availabilityZone: a
        routeTable: Network-Endpoints-Tgw-A
        ipv4CidrBlock: 10.1.3.208/28
      - name: Network-EndpointsTgwAttach-B
        availabilityZone: b
        routeTable: Network-Endpoints-Tgw-B
        ipv4CidrBlock: 10.1.3.224/28
    transitGatewayAttachments:
      - name: Network-Endpoints
        transitGateway:
          name: Network-Main
          account: Network
        routeTableAssociations:
          - Network-Main-Shared
        routeTablePropagations:
          - Network-Main-Core
          - Network-Main-Shared
          - Network-Main-Segregated
        subnets:
          - Network-EndpointsTgwAttach-A
          - Network-EndpointsTgwAttach-B
    gatewayEndpoints:
      defaultPolicy: Default
      endpoints:
        - service: s3
        - service: dynamodb
    interfaceEndpoints:
      central: true
      defaultPolicy: Default
      subnets:
        - Network-Endpoints-A
        - Network-Endpoints-B
      endpoints:
        - service: ec2
        - service: ec2messages
        - service: ssm
        - service: ssmmessages
        - service: kms
        - service: logs
        # - service: secretsmanager
        # - service: cloudformation
        # - service: access-analyzer
        # - service: application-autoscaling
        # - service: appmesh-envoy-management
        # - service: athena
        # - service: autoscaling
        # - service: autoscaling-plans
        # - service: clouddirectory
        # - service: cloudtrail
        # - service: codebuild
        # - service: codecommit
        # - service: codepipeline
        # - service: config
        # - service: datasync
        # - service: ecr.dkr
        # - service: ecs
        # - service: ecs-agent
        # - service: ecs-telemetry
        # - service: elasticfilesystem
        # - service: elasticloadbalancing
        # - service: elasticmapreduce
        # - service: events
        # - service: execute-api
        # - service: git-codecommit
        # - service: glue
        # - service: kinesis-streams
        # - service: kms
        # - service: logs
        # - service: monitoring
        # - service: sagemaker.api
        # - service: sagemaker.runtime
        # - service: servicecatalog
        # - service: sms
        # - service: sns
        # - service: sqs
        # - service: storagegateway
        # - service: sts
        # - service: transfer
        # - service: workspaces
        # - service: awsconnector
        # - service: ecr.api
        # - service: kinesis-firehose
        # - service: states
        # - service: acm-pca
        # - service: cassandra
        # - service: ebs
        # - service: elasticbeanstalk
        # - service: elasticbeanstalk-health
        # - service: email-smtp
        # - service: license-manager
        # - service: macie2
        # - service: notebook
        # - service: synthetics
        # - service: transfer.server
  - name: Network-Inspection
    account: Network
    region: *HOME_REGION
    cidrs:
      - 10.2.0.0/22
    routeTables:
      - name: Network-Inspection-Tgw-A
        routes: []
      - name: Network-Inspection-Tgw-B
        routes: []
      - name: Network-Inspection-A
        routes:
          - name: TgwRoute
            destination: 0.0.0.0/0
            type: transitGateway
            target: Network-Main
          - name: S3Gateway
            type: gatewayEndpoint
            target: s3
          - name: DynamoDBGateway
            type: gatewayEndpoint
            target: dynamodb
      - name: Network-Inspection-B
        routes:
          - name: TgwRoute
            destination: 0.0.0.0/0
            type: transitGateway
            target: Network-Main
          - name: S3Gateway
            type: gatewayEndpoint
            target: s3
          - name: DynamoDBGateway
            type: gatewayEndpoint
            target: dynamodb
      # - name: Network-Inspection-Tgw-A
      #   routes:
      #     - name: NfwRoute
      #       destination: 0.0.0.0/0
      #       type: networkFirewall
      #       target: accelerator-firewall
      #       targetAvailabilityZone: a
      #     - name: GwlbRoute
      #       destination: 0.0.0.0/0
      #       type: gatewayLoadBalancerEndpoint
      #       target: Endpoint-A
      # - name: Network-Inspection-Tgw-B
      #   routes:
      #     - name: NfwRoute
      #       destination: 0.0.0.0/0
      #       type: networkFirewall
      #       target: accelerator-firewall
      #       targetAvailabilityZone: b
      #     - name: GwlbRoute
      #       destination: 0.0.0.0/0
      #       type: gatewayLoadBalancerEndpoint
      #       target: Endpoint-B
    subnets:
      - name: Network-Inspection-A
        availabilityZone: a
        routeTable: Network-Inspection-A
        ipv4CidrBlock: 10.2.0.0/24
      - name: Network-Inspection-B
        availabilityZone: b
        routeTable: Network-Inspection-B
        ipv4CidrBlock: 10.2.1.0/24
      - name: Network-InspectionTgwAttach-A
        availabilityZone: a
        routeTable: Network-Inspection-Tgw-A
        ipv4CidrBlock: 10.2.3.208/28
      - name: Network-InspectionTgwAttach-B
        availabilityZone: b
        routeTable: Network-Inspection-Tgw-B
        ipv4CidrBlock: 10.2.3.224/28
      - name: Network-Inspection-Firewall-A
        availabilityZone: a
        routeTable: Network-Inspection-A
        ipv4CidrBlock: 10.2.3.0/28
      - name: Network-Inspection-Firewall-B
        availabilityZone: b
        routeTable: Network-Inspection-B
        ipv4CidrBlock: 10.2.3.16/28
    transitGatewayAttachments:
      - name: Network-Inspection
        transitGateway:
          name: Network-Main
          account: Network
        options:
          applianceModeSupport: enable
        routeTableAssociations:
          - Network-Main-Shared
        routeTablePropagations:
          - Network-Main-Core
          - Network-Main-Shared
          - Network-Main-Segregated
        subnets:
          - Network-InspectionTgwAttach-A
          - Network-InspectionTgwAttach-B
    gatewayEndpoints:
      defaultPolicy: Default
      endpoints:
        - service: s3
        - service: dynamodb
    useCentralEndpoints: true
  - name: SharedServices-Main
    account: SharedServices
    region: *HOME_REGION
    cidrs:
      - 10.4.0.0/16
    routeTables:
      - name: SharedServices-Tgw-A
        routes: []
      - name: SharedServices-Tgw-B
        routes: []
      - name: SharedServices-App-A
        routes:
          - name: TgwRoute
            destination: 0.0.0.0/0
            type: transitGateway
            target: Network-Main
          - name: S3Gateway
            type: gatewayEndpoint
            target: s3
          - name: DynamoDBGateway
            type: gatewayEndpoint
            target: dynamodb
      - name: SharedServices-App-B
        routes:
          - name: TgwRoute
            destination: 0.0.0.0/0
            type: transitGateway
            target: Network-Main
          - name: S3Gateway
            type: gatewayEndpoint
            target: s3
          - name: DynamoDBGateway
            type: gatewayEndpoint
            target: dynamodb
    subnets:
      - name: SharedServices-App-A
        availabilityZone: a
        routeTable: SharedServices-App-A
        ipv4CidrBlock: 10.4.0.0/24
      - name: SharedServices-App-B
        availabilityZone: b
        routeTable: SharedServices-App-B
        ipv4CidrBlock: 10.4.1.0/24
      - name: SharedServices-MainTgwAttach-A
        availabilityZone: a
        routeTable: SharedServices-Tgw-A
        ipv4CidrBlock: 10.4.255.208/28
      - name: SharedServices-MainTgwAttach-B
        availabilityZone: b
        routeTable: SharedServices-Tgw-B
        ipv4CidrBlock: 10.4.255.224/28
    transitGatewayAttachments:
      - name: SharedServices-Main
        transitGateway:
          name: Network-Main
          account: Network
        routeTableAssociations:
          - Network-Main-Shared
        routeTablePropagations:
          - Network-Main-Core
          - Network-Main-Shared
          - Network-Main-Segregated
        subnets:
          - SharedServices-MainTgwAttach-A
          - SharedServices-MainTgwAttach-B
    gatewayEndpoints:
      defaultPolicy: Default
      endpoints:
        - service: s3
        - service: dynamodb
    useCentralEndpoints: true

##############################################################
# Global configuration for VPC flow logs                     #
# Where there is no flow log configuration defined with VPC  #
# this configuration will be used for flow log configuration #
##############################################################
vpcFlowLogs:
  trafficType: ALL
  maxAggregationInterval: 600
  destinations:
    - s3
    - cloud-watch-logs
  defaultFormat: false
  customFields:
    - version
    - account-id
    - interface-id
    - srcaddr
    - dstaddr
    - srcport
    - dstport
    - protocol
    - packets
    - bytes
    - start
    - end
    - action
    - log-status
    - vpc-id
    - subnet-id
    - instance-id
    - tcp-flags
    - type
    - pkt-srcaddr
    - pkt-dstaddr
    - region
    - az-id
    - pkt-src-aws-service
    - pkt-dst-aws-service
    - flow-direction
    - traffic-path

#####################################
# Direct Connect Gateways-- use this #
# object to deploy DX Gateways,      #      
# virtual interfaces, and            #
# associations to transit gateways   #
#####################################
# directConnectGateways:
#   - name: Network-DXGW
#     account: Network
#     asn: 65000
#     gatewayName: Network-DXGW
#     virtualInterfaces:
#       - name: Accelerator-VIF
#         connectionId: dxcon-test1234
#         customerAsn: 65002
#         interfaceName: Accelerator-VIF
#         ownerAccount: Network
#         region: us-east-1
#         type: transit
#         vlan: 575
#         enableSiteLink: true
#         jumboFrames: true
#     transitGatewayAssociations:
#       - name: Network-Main
#         account: Network
#         allowedPrefixes:
#           - 10.0.0.0/8
#           - 192.168.0.0/16
#         routeTableAssociations:
#           - Network-Main-Core
#         routeTablePropagations:
#           - Network-Main-Core

#####################################
# VPC peering-- use this object to  #
# deploy peering between two VPCs   #
#####################################
# vpcPeering:
#   - name: VpcAtoVpcB
#     vpcs:
#       - VPC-A
#       - VPC-B
```
