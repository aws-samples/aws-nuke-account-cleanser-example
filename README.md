
# AWS Account Cleanser framework using aws-nuke

### AWS Nuke is an open source tool created by rebuy.de
### https://github.com/rebuy-de/aws-nuke
### AWS Nuke searches for deleteable resources in the provided AWS acccount and deletes those which are not considered "Default" or "AWS-Managed"
### In short, it will take your account back to Day1 with few exceptions

```sh
$ WARNING: \
This is a very destructive tool and should not be deployed without fully understanding the impact it will have on the AWS accounts you allow it to interface with. \
The aws-nuke binary if not configured correctly, can delete unintended resources in your AWS accounts. \
Please use caution and configure this tool to delete unused resources only in your lower test/sandbox environment accounts.
```

## Overview

The code in this repository helps you set up the following target architecture.

![infrastructure-overview](images/architecture-overview.png)

* The approach covered in this pattern is suitable for customers needing an automated mechanism to clean up their obsolete resources from test or sandbox  accounts periodically.  It is quite common that customers will have a set of Dev/Sandbox accounts where developers can create and experiment with various services and resources, which are then left unattended or obsolete, and can quickly lead to high and unnecessary AWS Cost/Spending ( example: Developers created an expensive DB instance ( RDS) or EBS/EFS volume for testing and missed to terminate those resources ).


* This solution sets up an automated mechanism using the binary aws-nuke along with AWS Step Functions , EventBridge and AWS CodeBuild which can run on a daily scheduled basis to scan and delete the resources from the Sandbox account across each region in a scalable manner. A CodeBuild project is  invoked from the Step Functions map state for each region of an account to delete resources specific to that region, thus providing scalability and better control in terms of time and monitoring efficiency.

    This architecture provides the following features:

    1. The workflow is kicked off based off a scheduled event trigger set up in AWS EventBridge which invokes the Step Functions.
    2. The orchestration of this pattern using Step Functions serves the purpose of handling resources in each region using a separate invocation of CodeBuild Project ( the region attribute in the nuke config  file required by the aws-nuke binary is dynamically updated using a custom python class inside the CodeBuild project )  with the region parameter and the customized nuke config , thus providing dynamic parallelism with the Map state that fans out for all the regions as needed within the sandbox account and thus saves time and provide scalability to handle a lot of resources.
    3. Generally the aws-nuke command line need to assume STS temporary credentials for handling the resources in each account using IAM Role-chaining which limits it's max time to 60 minutes most of the times and hence if multiple regions are configured to run in one CodeBuild project execution , there could be lot of stale resources still left out without being destroyed.
        - This pattern takes care of that by executing the clean up for each region in a single account in parallel using the Step Functions Map state.
        - This also uses the credentials for the aws-nuke binary configured with the profile, that can be configured with the shared aws config file ( ~/.aws/config) which doesn't expire with a 1 hour session limit.
        - Also this increases the default CodeBuild project time out to about 2-4 hours , allowing more time for the nuke to complete deleting all resources for each reg   
    4. The aws-nuke binary works based off the nuke-config.yaml file , which is dynamically updated in this pattern using a python filtering class , to provide flexibility in handling the resource filters and region constraints based on the supplied override parameters.
    5. This workflow invokes the CodeBuild project synchronously and waits for Success. It also has a retry mechanism to trigger the CodeBuild project again with a configured amount of time , if in case it errors out, making sure all the resources are handled in the daily run without any manual intervention.
    6. The workflow also sends out a detailed report to an SNS topic subscription on what resources were deleted after the job is successful for each region which simplifies traversing and parsing the complex logs written out by the aws-nuke binary.

## Prerequisites 

1. https://github.com/rebuy-de/aws-nuke --> Open source library staged/downloaded to artifactory or S3. This aws-nuke binary is owned by rebuy-de

2. AWS Account alias needs to exist in the IAM Dashboard for the target sandbox account for 'aws-nuke' to work

3. AWS CodeBuild project --> for runtime/compute

4. AWS S3 Bucket --> for storing the nuke config file and the aws-nuke binary ( latest version ) if needed. Make sure to source the latest 'aws-nuke' binary downloaded in S3 (or from your internal artifactory)

5. AWS StepFunctions --> For orchestration for fan out of multi-region parallel CodeBuild invocation targets

6. AWS SNS Topic  -->  An active email address with SNS topic subscription to send the CodeBuild job status and the detailed report of resources nuked daily

7. AWS EventBridge Rule --> Configured with the required cron schedule to trigger the workflow periodically. The input parameters to invoke the Rule target should be updated with the required region lists as needed

8. Make sure you have sufficient network connectivity from the VPC where this is run, as CodeBuild downloads the nuke binary from github. If running in restricted environment, have the binary uploaded to S3 bucket or artifactory and reference that.

## Design

* CodeBuild provides a super-handy way for us to spin up a container and run a script without having to worry about provisioning and maintaining resources. We’re using CloudFormation to define the whole project, so the code sample below displays the bulk of the Project resource configuration for a CloudFormation template. In short, we’re building a standard AWS Linux Docker container, configuring the log output channel, assigning an AWS Role for the nuke binary to assume the account roles (mentioned earlier) to start the clean up based on the region and config file.

* AWS EventBridge Events provides both event-based and scheduled triggers for automated actions in other services. This is basically a fancy Cron job to schedule the build project. As with the CodeBuild Project, the example below is a resource defined within a CloudFormation template. In short, it defines the schedule expression for the Cron job (3:00a EST Mon-Fri), a role to allow the event trigger to run and kick off the StepFunctions workflow as a target which will orchestrate the CodeBuild project invocation based on the supplied region list parameter and the dynamic nuke config file modified during runtime.

* AWS StepFunctions provides this design with scalability and help achieve dynamic parallelism across accounts/regions using the Map State. The orchestration of this pattern using Step Functions serves the purpose of handling resources in each region using a separate invocation of CodeBuild Project ( the region attribute in the nuke config  file required by the aws-nuke binary is dynamically updated using a custom python class inside the CodeBuild project )  with the region parameter and the customized nuke config , thus providing dynamic parallelism with the Map state that fans out for all the regions as needed within the sandbox account and thus saves time and provide scalability to handle a lot of resources.

## Dry Runs vs Production

By default, this script will not take any destructive action on any resources in your account(s). It will provide a log of the “dry run” output as if it actually completed the actions specified. When you’ve thoroughly tested this and whitelisted any resources in your own aws-nuke-config.yaml, you need to add the –-no-dry-run flag to the aws-nuke command in this script to force a destructive run.

```sh
$ aws-nuke -c $line.yaml --force --no-dry-run --access-key-id $ACCESS_KEY_ID --secret-access-key $SECRET_ACCESS_KEY --session-token $SESSION_TOKEN |tee -a aws-nuke.log;
```

## Setup and Installation

* Clone the repo
* Determine the ID of the account to be deployed for clean up ( This is only to be deployed to Dev/Test/Sandbox environments )
* Verify and Update your nuke configuration file as needed with specific filters for the resources/accounts
* Deploy the stack using the below command. You can run it in any desired region.
```sh
aws cloudformation create-stack --stack-name NukeCleanser --template-body file://nuke-cfn-stack.yaml --region us-east-2 --capabilities CAPABILITY_NAMED_IAM
```
* Once the stack is created, upload the nuke generic config file and the python script to the S3 bucket using the commands below. You can find the name of the S3 bucket generated from the CloudFormation console `Outputs` tab.
```sh
aws s3 cp config/nuke_generic_config.yaml --region us-east-2 s3://{your-bucket-name}
aws s3 cp config/nuke_config_update.py --region us-east-2 s3://{your-bucket-name}
```
* Run the stack manually by triggering the StepFunctions with the below sample input payload. (which is pre-configured in the EventBridge Target as a Constant JSON input). You can configure this to run in parallel on the required number of regions by updating the region_list parameter.

```sh
{
  "InputPayLoad": {
    "nuke_dry_run": "true",
    "nuke_version": "2.21.2",
    "region_list": [
      "global",
      "us-west-1",
      "us-east-1"
    ]
  }
}
```

* The tool is currently configured to run at a schedule as desired typically off hours 3:00a EST. It can be easily configured with a rate() or cron() expression by editing the cfn template file

* The workflow also sends out a detailed report to an SNS topic with an active email subscription on what resources were deleted after the job is successful for each region which simplifies traversing and parsing the complex logs spit out by the aws-nuke binary. 

* If the workflow is successful , the stack will send out
  - One email for each of the regions where nuke CodeBuild job was invoked with details of the build execution , the list of resources which was deleted along with the log file path. 
  - The StepFunctions workflow also sends out another email when the whole Map state process completes successfully. Sample email template given below.

```sh
Account Cleansing Process Completed;

  ------------------------------------------------------------------
  Summary of the process:
  ------------------------------------------------------------------
  DryRunMode                   : true
  Account ID                   : 123456789012
  Target Region                : us-west-1
  Build State                  : JOB SUCCEEDED
  Build ID                     : AccountNuker-NukeCleanser:4509a9b5
  CodeBuild Project Name       : AccountNuker-NukeCleanser
  Process Start Time           : Thu Feb 23 04:05:21 UTC 2023
  Process End Time             : Thu Feb 23 04:05:54 UTC 2023
  Log Stream Path              : AccountNuker-NukeCleanser/logPath
  ------------------------------------------------------------------
  ################ Nuke Cleanser Logs #################

  FAILED RESOURCES
-------------------------------
Total number of Resources that would be removed:
3
us-west-1 - SQSQueue - https://sqs.us-east-1.amazonaws.com/123456789012/test-nuke-queue - would remove
us-west-1 - SNSTopic - TopicARN: arn:aws:sns:us-east-1:123456789012:test-nuke-topic - [TopicARN: "arn:aws:sns:us-east-1:123456789012:test-topic"] - would remove
us-west-1 - S3Bucket - s3://test-nuke-bucket-us-west-1 - [CreationDate: "2023-01-25 11:13:14 +0000 UTC", Name: "test-nuke-bucket-us-west-1"] - would remove

```
* By default the stack runs aws-nuke in DryRun mode, To actually delete resources update the stack with AWSNukeDryRunFlag parameter flipped to false OR udpate manually in the CodeBuild environment variables section.

## Monitoring queries

* Using aws-cli cloudwatch logs

```sh
aws logs filter-log-events \
  --log-group-name AccountNuker-nuke-auto-account-cleanser \
  --start-time 1628838256000 --end-time 1628839216000 \
  --log-stream-names "10409c89-a90f-4af7-9642-0df9bc9f0855" \
  --filter-pattern removed \
  --no-interleaved \
  --output text \
  --limit 5
```

* Using awslogs for analyzing output from aws-nuke runs

```sh
awslogs get AccountNuker-nuke-auto-account-cleanser --filter-pattern '"Scan complete: "' --start='1d ago' --timestamp
awslogs get AccountNuker-nuke-auto-account-cleanser --filter-pattern '"Error: failed"' --start='1d ago' | sort -u
awslogs get AccountNuker-nuke-auto-account-cleanser --filter-pattern '"Removal requested: 0 waiting"' --start='1d ago' | sort -u
awslogs get AccountNuker-nuke-auto-account-cleanser --filter-pattern '"AccessDenied"' --start='1d ago' | sort -u | wc -l
```


* Using CW Logs Insights query

```sh
fields @timestamp, @message
| filter userIdentity.sessionContext.sessionIssuer.userName = "nuke-auto-account-cleanser" and ispresent(errorCode) 
| sort @timestamp desc
| limit 500


fields @timestamp, @message
| filter ispresent(errorCode) and userIdentity.sessionContext.sessionIssuer.userName = "nuke-auto-account-cleanser"
and errorCode != "AccessDenied" and eventName like "Delete"
| sort @timestamp desc
| limit 500


fields @timestamp, @message
| filter ispresent(errorCode) and userIdentity.sessionContext.sessionIssuer.userName = "nuke-auto-account-cleanser"
and errorCode == "AccessDenied" and eventName like "Delete"
| sort @timestamp desc
| limit 500
```

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

