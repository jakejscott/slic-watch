# slic-watch

[![serverless](http://public.serverless.com/badges/v3.svg)](http://www.serverless.com)
[![npm version](https://img.shields.io/npm/v/serverless-slic-watch-plugin)](https://npm.im/serverless-slic-watch-plugin)
[![Build](https://github.com/fourTheorem/slic-watch/actions/workflows/build.yml/badge.svg)](https://github.com/fourTheorem/slic-watch/actions/workflows/build.yml)
[![Coverage Status](https://coveralls.io/repos/github/fourTheorem/slic-watch/badge.svg)](https://coveralls.io/github/fourTheorem/slic-watch)
[![JavaScript Style Guide](https://img.shields.io/badge/code_style-standard-brightgreen.svg)](https://standardjs.com)


SLIC Watch provides a CloudWatch Dashboard and Alarms for:

 1. AWS Lambda
 2. API Gateway
 3. DynamoDB
 4. Kinesis Data Streams
 5. SQS Queues
 6. Step Functions

Currently, SLIC Watch is available as a Serverless Framework plugin.

## Installation
```
npm install serverless-slic-watch-plugin --save-dev
```

## Features

CloudWatch Alarms and Dashboard widgets are created for all supported resources in the CloudFormation stack generated by The Serverless Framework. This includes generated resources as well as resources specifed explicitly in the `resources` section.
Any feature can be configured or disabled completely - see the section on [configuration](#configuration) to see how.

### Lambda Functions

Lambda Function alarms are created for:
1. _Errors_
2. _Throttles_, as a percentage of the number of invocations
3. _Duration_, as a percentage of the function's configured timeout
4. _Invocations_, disabled by default
5. _IteratorAge_, for function's triggered by an [Event Source Mapping](https://docs.aws.amazon.com/lambda/latest/dg/invocation-eventsourcemapping.html)

Lambda dashboard widgets show:

|Errors|Throttles|Duration Average, P95 and Maximum|
|--|--|--|
|![Errors](/docs/lambdaErrors.png)|![Throttles](/docs/lambdaThrottles.png) |![Throttles](/docs/lambdaDurationP95.png) |
|**Invocations**|**Concurrent Executions**|**Iterator Age**|
|![Invocations](/docs/lambdaInvocations.png) |![concurrent executions](/docs/lambdaConcurrentExecutions.png) |![Iterator Age](/docs/lambdaIteratorAge.png) |

### API Gateway

API Gateway alarms are created for:
1. 5XX Errors
2. 4XX Errors
3. Latency

API Gateway dashboard widgets show:

|5XX Errors|4XX Errors|Latency|Count|
|--|--|--|--|
|![5XX Errors](/docs/api5xx.png)|![4XX Errors](/docs/api4xx.png) |![Latency](/docs/apiLatency.png) |![Count](/docs/apiCount.png) |

### DynamoDB

DynamoDB alarms are created for:
1. Read Throttle Events (Table and GSI)
2. Write Throttle Events (Table and GSI)
3. UserErrors
4. SystemErrors

Dashboard widgets are created for tables and GSIs:
dynamodbGSIReadThrottle.png    dynamodbGSIWriteThrottle.png     dynamodbTableWriteThrottle.png

|ReadThrottleEvents (Table)| WriteThrottleEvent (Table)|
|---|---|
|![WriteThrottleEvents Table](/docs/dynamodbTableWriteThrottle.png)|![WriteThrottleEvents Table](/docs/dynamodbTableWriteThrottle.png)|
|**ReadThrottleEvents (GSI)**|**WriteThrottleEvent (GSI)**|
|![WriteThrottleEvents GSI](/docs/dynamodbGSIWriteThrottle.png)|![WriteThrottleEvents GSI](/docs/dynamodbGSIWriteThrottle.png)|

### Kinesis Data Streams
Kinesis data stream alarms are created for:
1. Iterator Age
2. Read Provisioned Throughput Exceeded
2. Write Provisioned Throughput Exceeded
3. PutRecord.Success
3. PutRecords.Success
4. GetRecords.Success

Kinesis data stream dashboard widgets show:

|Iterator Age|Read Provisioned Throughput Exceeded|Write Provisioned Throughput Exceeded|
|--|--|--|
|![Iterator Age](/docs/kinesisIteratorAge.png)|![Provisioned Throughput Exceeded](/docs/kinesisProvisionedThroughput.png) |![Put/Get Success](/docs/kinesisPutGetSuccess.png) |

### SQS Queues

SQS Queue alarms are create for:
1. Age Of Oldest Message (disabled by default). If enabled, a threshold in seconds should be specified.
2. In Flight Messages Percentage. This is a percentage of the [AWS hard limits](https://aws.amazon.com/sqs/faqs/) (20,000 messages for FIFO queues and 120,000 for standard queues).

SQS queue dashboard widgets show:

|Messages Sent, Received and Deleted|Messages Visible|Age of Oldest Message|
|--|--|--|
|![Messages](/docs/sqsMessages.png)|![Messages Visible](/docs/sqsMessagesInQueue.png) |![Oldest Message](/docs/sqsOldestMessage.png) |

### Step Functions

Step Function alarms are created for:
1. Execution Throttled
2. Executions Failed
3. Executions Timed Out

The dashboard contains one widget per Step Function:
|ExecutionsFailed ExecutionThrottled, ExecutionsTimedOut|
|--|
|![Step Function widget](/docs/stepFunctions.png)

## Configuration

The `topic` configuration must be configured with the ARN of an SNS Topic.
Alarm configuration is _cascading_. This means that configuration properties are automatically propagated from parent to children nodes (unless an override is present at the given node).
Supported options along with their defaults are shown below.


```yaml
# ...

custom:
  slicWatch:
    topic: SNS_TOPIC_ARN

    alarms:
      enabled: true
      Period: 60
      EvaluationPeriods: 1
      TreatMissingData: notBreaching
      ComparisonOperator: GreaterThanThreshold
      Lambda: # Lambda Functions
        Errors:
          Threshold: 0
          Statistic: Sum
        ThrottlesPc: # Throttles are evaluated as a percentage of invocations
          Threshold: 0
        DurationPc: # Duration is evaluated as a percentage of the function timeout
          Threshold: 95
          Statistic: Maximum
        Invocations: # No invocation alarms are created by default. Override threshold to create alarms
          enabled: false # Note: this one requires both `enabled: true` and `Threshold: someValue` to be effectively enabled
          Threshold: null
          Statistic: Sum
        IteratorAge:
          Threshold: 10000
          Statistic: Maximum
      ApiGateway: # API Gateway REST APIs
        5XXError:
          Statistic: Average
          Threshold: 0
        4XXError:
          Statistic: Average
          Threshold: 0.05
        Latency:
          ExtendedStatistic: p99
          Threshold: 5000
      States: # Step Functions
        Statistic: Sum
        ExecutionsThrottled:
          Threshold: 0
        ExecutionsFailed:
          Threshold: 0
        ExecutionsTimedOut:
          Threshold: 0
      DynamoDB:
        # Consumed read/write capacity units are not alarmed. These should either
        # be part of an auto-scaling configuration for provisioned mode or should be automatically
        # avoided for on-demand mode. Instead, we rely on persistent throttling
        # to alert failures in these scenarios.
        # Throttles can occur in normal operation and are handled with retries. Threshold should
        # therefore be configured to provide meaningful alarms based on higher than average throttling.
        Statistic: Sum
        ReadThrottleEvents:
          Threshold: 10
        WriteThrottleEvents:
          Threshold: 10
        UserErrors:
          Threshold: 0
        SystemErrors:
          Threshold: 0
      Kinesis:
        GetRecords.IteratorAgeMilliseconds:
          Statistic: Maximum
          Threshold: 10000
        ReadProvisionedThroughputExceeded:
          Statistic: Maximum
          Threshold: 0
        WriteProvisionedThroughputExceeded:
          Statistic: Maximum
          Threshold: 0
        PutRecord.Success:
          ComparisonOperator: LessThanThreshold
          Statistic: Average
          Threshold: 1
        PutRecords.Success:
          ComparisonOperator: LessThanThreshold
          Statistic: Average
          Threshold: 1
        GetRecords.Success:
          ComparisonOperator: LessThanThreshold
          Statistic: Average
          Threshold: 1
        SQS:
          # approximate age of the oldest message in the queue above threshold: messages aren't processed fast enough
          AgeOfOldestMessage:
            Statistic: Maximum
            enabled: false # Note: this one requires both `enabled: true` and `Threshold: someValue` to be effectively enabled
            Threshold: null
          # approximate number of messages in flight above threshold (in percentage of hard limit: 120000 for regular queues and 20000 for FIFO queues)
          InFlightMessagesPc:
            Statistic: Maximum
            Threshold: 80 # 80% of 120.000 for regular queues or 80% of 20000 for FIFO queues

    dashboard:
      timeRange:
        # For possible 'start' and 'end' values, see
        # https:# docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/CloudWatch-Dashboard-Body-Structure.html
        start: -PT3H
      metricPeriod: 300
      widgets:
        metricPeriod: 300
        width: 8
        height: 6
        Lambda:
          # Metrics per Lambda Function
          Errors:
            Statistic: ['Sum']
          Throttles:
            Statistic: ['Sum']
          Duration:
            Statistic: ['Average', 'p95', 'Maximum']
          Invocations:
            Statistic: ['Sum']
          ConcurrentExecutions:
            Statistic: ['Maximum']
          IteratorAge:
            Statistic: ['Maximum']
        ApiGateway:
          5XXError:
            Statistic: ['Sum']
          4XXError:
            Statistic: ['Sum']
          Latency:
            Statistic: ['Average', 'p95']
          Count:
            Statistic: ['Sum']
        States:
          # Step Functions
          ExecutionsFailed:
            Statistic: ['Sum']
          ExecutionsThrottled:
            Statistic: ['Sum']
          ExecutionsTimedOut:
            Statistic: ['Sum']
        DynamoDB:
          # Tables and GSIs
          ReadThrottleEvents:
            Statistic: ['Sum']
          WriteThrottleEvents:
            Statistic: ['Sum']
        Kinesis:
          # Kinesis Data Streams
          GetRecords.IteratorAgeMilliseconds:
            Statistic: ['Maximum']
          ReadProvisionedThroughputExceeded:
            Statistic: ['Sum']
          WriteProvisionedThroughputExceeded:
            Statistic: ['Sum']
          PutRecord.Success:
            Statistic: ['Average']
          PutRecords.Success:
            Statistic: ['Average']
          GetRecords.Success:
            Statistic: ['Average']
        SQS:
          # SQS Queues
          NumberOfMessagesSent:
            Statistic: ["Sum"]
          NumberOfMessagesReceived:
            Statistic: ["Sum"]
          NumberOfMessagesDeleted:
            Statistic: ["Sum"]
          ApproximateAgeOfOldestMessage:
            Statistic: ["Maximum"]
          ApproximateNumberOfMessagesVisible:
            Statistic: ["Maximum"]
```

An example project is provided for reference: [serverless-test-project](./serverless-test-project)

## A note on CloudWatch cost

This plugin creates additional CloudWatch resources that, apart from a limited free tier, have an associated cost.
Depending on what you enable, SLIC Watch creates one dashboard and multiple alarms. The number of each depend on the number of resources in your stack and the number of stacks you have.

Check out the AWS [CloudWatch Pricing](https://aws.amazon.com/cloudwatch/pricing/) page to understand the cost impact of creating CloudWatch resources.

## Releasing a new version

In order to release a new version of the project:

  - update the package version in the top level `package.json`
  - run `npm run lerna:sync` to synchronise that version across all the sub packages
  - push these changes
  - draft a new release in GitHub (the CI will do the publish to npm)

## References

### Other Projects

1. [serverless-plugin-aws-alerts](https://www.npmjs.com/package/serverless-plugin-aws-alerts)
2. [Real World Serverless Application - Serverless Operations](https://github.com/awslabs/realworld-serverless-application/wiki/Serverless-Operations)
3. [CDK Patterns - The CloudWatch Dashboard](https://github.com/cdk-patterns/serverless/blob/main/the-cloudwatch-dashboard/README.md)

### Reading

1. [AWS Well Architected Serverless Applications Lens](https://docs.aws.amazon.com/wellarchitected/latest/serverless-applications-lens/welcome.html)
2. [How to Monitor Lambda with CloudWatch Metrics - Yan Cui](https://lumigo.io/serverless-monitoring-guide/how-to-monitor-lambda-with-cloudwatch-metrics/)

## LICENSE

Apache - [LICENSE](./LICENSE)
