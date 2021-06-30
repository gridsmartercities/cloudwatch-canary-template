[CloudWatch Canaries](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Synthetics_Canaries.html) can monitor your sites and apis, while [CloudFormation](https://aws.amazon.com/cloudformation/) automates the deployment of AWS resources. In this post, I will show you how these two technologies can work together to automate the deployment of these canaries to monitor a range of applications. 


## **Why CloudFormation?**

At [Grid Smarter Cities](https://github.com/gridsmartercities) we use CloudFormation to automate the deployment of a wide range of stacks, including:



*   CI/CD processes for our Front End and Back End applications
*   Management of users and permissions
*   Deployment of tools like CloudWatch Canaries and Lambda Performance Tuning

We do this because it makes the process of automatically deploying resources efficient and less time consuming. CloudFormation treats infrastructure as code, meaning both resources and their dependencies can be created, validated and deployed from a single template file. 

If the stack needs to be updated, all the user has to do is make the necessary changes to the template, reupload it, and let CloudFormation verify and make all the necessary changes to your resources. If a stack needs duplicating, upload the template again and adjust the parameters and redeploy. If the stack is no longer needed, everything can be deleted with a single button click. 


## **Resources Required Separate To the Stack**

Before the stack is deployed, we will need to have a prepared script for the canary and two S3 Buckets. The first bucket is used by the Canary to store its artefacts saved, while it  is running - this could be screenshots, logs or generated reports. The second bucket will be used to store the source code used by the canary - we are using a Node script for this canary. The code for this can be found [here](https://github.com/gridsmartercities/cloudwatch-canary-template).


### **Uploading the Files**

These files need to be in a particular folder structure - this wasn't clear on the AWS CloudFormation Documentation. However, I eventually found that your canary's code file structure should be /nodejs/node_modules/ .

The following two commands will create this folder structure, and prepare the zip folder: 

```
cp canary-function.js ./nodejs/node_modules/
zip -r sourcecode.zip ./nodejs/*
```
The new sourcecode.zip file just needs to be uploaded directly into the S3 bucket you are using to store the Canary source code.


## **Writing The Template**

In this template, we have added parameters for:

*   The name of the Service we're monitoring
*   The name of the Canary
*   The name of Source Code Bucket
*   The name of the object containing the code inside the SourceCode bucket
*   The email address for alert emails to be sent to if the canary fails 

All of these values will be inputted by the user and passed as parameters when the user has uploaded the template. 

### **MetaData**
```
Metadata:
  AWS::CloudFormation::Interface:
    ParameterGroups:
      - Label:
          default: 'Service'
        Parameters:
          - ServiceName
          - CanaryName
          - AlertEmail
      - Label:
          default: 'Buckets'
        Parameters:
          - CanarySourceCodeBucketName
          - CanarySourceCodeKey
          - CanaryArtifactBucketName
    ParameterLabels:
      ServiceName:
        default: Service Name
      CanaryName:
        default: Canary Name
      AlertEmail:
        default: Email
      CanarySourceCodeBucketName:
        default: Source code bucket name
      CanarySourceCodeKey:
        default: Key for the object 
      CanaryArtifactBucketName:
        default: The name of the S3 bucket where the artefacts will be store
```
### **Parameters**
```
Parameters:
  ServiceName:
    Description: Enter a lower case, high level service name without environment details. Used to autofill service names. For example, your-service-name
    Type: String
  CanaryName:
    Description: Enter a lower case name for the canary, as they should be known. For example dave not Dave
    Type: String
  AlertEmail:
    Description: Email address to send staging build alerts to, or example you@example.com
    Type: String
  CanarySourceCodeBucketName:
    Description: S3 Bucket Name where the source code lives
    Type: String
  CanarySourceCodeKey:
    Description: Key of the object which contains the source code
    Type: String
  CanaryArtifactBucketName:
    Description: S3 Bucket Name where the canary saves the screenshots and other 
    Type: String
```

### **Resources**

The resources we need are: 

*   A CloudWatch Canary
*   A Role for the CloudWatch Canary
*   A policy for the CloudWatch Canary's Role
*   A CloudWatch alert to detect when the Canary fails
*   An SNS topic which will send the email alerts

#### **CloudWatch Canary**

The first resource is of type AWS::Synthetics::Canary. The property specifying the canary's name and relevant S3 buckets have been read from the parameters defined earlier in the template. For now, the runtime version, handler and schedule have been hard coded, but these could easily be extracted as parameters to make the template more flexible.

```
CloudwatchCanary: 
    Type: AWS::Synthetics::Canary
    Properties: 
        ArtifactS3Location: !Sub s3://${CanaryArtifactBucketName}
        Code: 
            Handler: canary-function.handler
            S3Bucket: !Sub ${CanarySourceCodeBucketName}
            S3Key:  !Sub ${CanarySourceCodeKey}
        Name: dave-the-canary
        RuntimeVersion: syn-nodejs-puppeteer-3.1
        Schedule: 
            Expression: rate(5 minutes)
        StartCanaryAfterCreation: true
        ExecutionRoleArn: !GetAtt CloudwatchCanaryRole.Arn
```

#### **CloudWatch Canary Policy and CloudWatch Canary Role**

The second and third resources are of type AWS::IAM::Policy and AWS::IAM::Policy. We chose to give the CloudWatch Canary a policy which had been modified from [CloudWatchSyntheticsFullAccess](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Synthetics_Canaries_Roles.html). This gave our canary permission to: 

*   Read data from the source code buckets
*   Save and retrieve data from the artefact bucket
*   Trigger alarms when the canary detects an error.

We had to add a few extra actions and modify the resource selectors to match our resource names. There were also some unnecessary actions which - for best practices and security - were removed.

### **CloudWatch Canary Policy**
```
CloudwatchCanaryPolicy:
  Type: AWS::IAM::Policy
  Properties:
    PolicyName: !Sub ${ServiceName}-dave-the-canary-policy
    PolicyDocument:
    Version: '2012-10-17'
    Statement:
    - Effect: Allow
        Action:
        - synthetics:*
        Resource: "*"
    - Effect: Allow
        Action:
        - s3:CreateBucket
        - s3:PutEncryptionConfiguration
        Resource:
        - arn:aws:s3:::*
    - Effect: Allow
        Action:
        - iam:ListRoles
        - s3:ListAllMyBuckets
        - s3:GetBucketLocation
        - xray:GetTraceSummaries
        - xray:BatchGetTraces
        - apigateway:GET
        Resource: "*"
    - Effect: Allow
        Action:
        - s3:GetObject
        - s3:ListBucket
        - s3:PutObject
        Resource: arn:aws:s3:::*
    - Effect: Allow
        Action:
        - s3:GetObjectVersion
        Resource: arn:aws:s3:::*
    - Effect: Allow
        Action:
        - iam:PassRole
        Resource:
        - !Sub arn:aws:iam::*:role/service-role/${CloudwatchCanaryRole}
        Condition:
        StringEquals:
            iam:PassedToService:
            - lambda.amazonaws.com
            - synthetics.amazonaws.com
    - Effect: Allow
        Action:
        - iam:GetRole
        Resource:
        - !Sub arn:aws:iam::*:role/service-role/*
    - Effect: Allow
        Action:
        - cloudwatch:GetMetricData
        - cloudwatch:GetMetricStatistics
        Resource: "*"
    - Effect: Allow
        Action:
        - cloudwatch:PutMetricAlarm
        - cloudwatch:PutMetricData
        - cloudwatch:DeleteAlarms
        Resource:
        - '*'
    - Effect: Allow
        Action:
        - cloudwatch:DescribeAlarms
        Resource:
        - arn:aws:cloudwatch:*:*:alarm:*
    - Effect: Allow
        Action:
        - lambda:CreateFunction
        - lambda:AddPermission
        - lambda:PublishVersion
        - lambda:UpdateFunctionConfiguration
        - lambda:GetFunctionConfiguration
        Resource:
        - arn:aws:lambda:*:*:function:cwsyn-*
    - Effect: Allow
        Action:
        - lambda:GetLayerVersion
        - lambda:PublishLayerVersion
        Resource:
        - arn:aws:lambda:*:*:layer:cwsyn-*
        - arn:aws:lambda:*:*:layer:Synthetics:*
    - Effect: Allow
        Action:
        - ec2:DescribeVpcs
        - ec2:DescribeSubnets
        - ec2:DescribeSecurityGroups
        Resource:
        - "*"
    - Effect: Allow
        Action:
        - sns:ListTopics
        Resource:
        - "*"
    - Effect: Allow
        Action:
        - sns:CreateTopic
        - sns:Subscribe
        - sns:ListSubscriptionsByTopic
        Resource:
        - arn:*:sns:*:*:Synthetics-*
    Roles:
    - !Ref CloudwatchCanaryRole
```

## **CloudWatch Canary Role**
```
CloudwatchCanaryRole:
    Type: AWS::IAM::Role
    Properties:
        RoleName:  !Sub ${ServiceName}-${CanaryName}-the-canary-role
        AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
            - Effect: Allow
            Principal:
                Service:
                - lambda.amazonaws.com
            Action:
                - sts:AssumeRole
```


On its own, the deployed canary will monitor resources as canary's source code files. Any issues found by the canary will be visible on the relevant  dashboard in CloudWatch. Unfortunately, this will not trigger any email alerts or notifications. Adding this functionality to our stack is straightforward, we need to add a CloudWatchAlarm and an SNS Topic. 


### **Adding the Topic and Alarm**

By adding the following sections, alert emails will be sent to the specified email address when submitting the template. 

The CloudWatchCanaryAlarm is triggered when the average success count falls below 100% for one evaluation period; this period was hard coded to five minutes. If this failure is detected, the CloudWatchCanaryAlarmTopic is triggered.

```
CloudwatchCanaryAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
        ActionsEnabled: true
        AlarmDescription: !Sub ${ServiceName}-${CanaryName}-the-canary-alert
        ComparisonOperator:  LessThanThreshold
        EvaluationPeriods: 1
        DatapointsToAlarm: 1
        MetricName: SuccessPercent
        Namespace: CloudWatchSynthetics
        Period: 60
        Statistic: Average
        Threshold: 100
        TreatMissingData: notBreaching
        AlarmActions:
        - !Ref CloudwatchCanaryAlarmTopic
        Dimensions:
            - Name: CanaryName
            Value: !Sub ${CanaryName}-the-canary
```

The CloudWatchCanaryAlarmTopic sends a message to the AlertEmail address, specified by the CloudFormation parameters inputted by the user when deploying the stack.


## **Deploying Stack**

All that's left to do is deploy Dave the Canary. To do this,  upload the template, fill in the form defining the stack parameters as shown below, click next(we'll not configure any of the default stack options), click next again, confirm that you "acknowledge that AWS CloudFormation might create IAM resources with custom names" then click the Create Stack button. Your resources should then be deployed and running.

![Screenshot of completed form to set CloudFormation parameters](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/tgyqwa0swoocv9c7yeqt.png)
 
## **Closing Notes**

This CloudFormation template was written to help developers at Grid Smarter Cities deploy tools which monitor our products. Deploying a canary from the console is straightforward but using CloudFormation allows us to automate the deployment process, allowing us to set up monitoring tools and alerts for all of our products. 

Writing this template may have required a bit of research and some trial and error but, in doing so, our developers and QAs will never have to manually check that our products are live. If the products aren't, the CloudWatch canary will automatically inform us quickly and more reliably than if we were manually checking ourselves.

Full copies of both templates (with and without the alerts) and example canary function can be found in the Repo.
