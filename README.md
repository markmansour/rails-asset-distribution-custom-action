The custom action will be managed by an Elastic Beanstalk worker environment, running as a periodic process.

# Create a Rails Application to process AWS CodePipeline Jobs
$ rails new rails-asset-distribution-custom-action -O # skips active record

# Setup development env
For development purposes, allow requests to be served from port 80 on EC2 instace, which has the port unblocked

$ sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 3000

## Start the server
$ rails server --daemon --binding=`curl --silent http://169.254.169.254/latest/meta-data/local-ipv4`

## Stop the server
$ kill $(cat tmp/pids/server.pid)


# Create IAM instance policy for Elastic Beanstalk
## Create New Policy
Policy Name: aws-elastic-beanstalk-instance-profile

## Create New Role
Role Name:   aws-elasticbeanstalk-instance-profile-role
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "QueueAccess",
            "Action": [
                "sqs:ChangeMessageVisibility",
                "sqs:DeleteMessage",
                "sqs:ReceiveMessage",
                "sqs:SendMessage"
            ],
            "Effect": "Allow",
            "Resource": "*"
        },
        {
            "Sid": "MetricsAccess",
            "Action": [
                "cloudwatch:PutMetricData"
            ],
            "Effect": "Allow",
            "Resource": "*"
        },
        {
            "Sid": "BucketAccess",
            "Action": [
                "s3:Get*",
                "s3:List*",
                "s3:PutObject"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::elasticbeanstalk-*-{{accountid}}/*",
                "arn:aws:s3:::elasticbeanstalk-*-{{accountid}}-*/*"
            ]
        },
        {
            "Sid": "DynamoPeriodicTasks",
            "Action": [
                "dynamodb:BatchGetItem",
                "dynamodb:BatchWriteItem",
                "dynamodb:DeleteItem",
                "dynamodb:GetItem",
                "dynamodb:PutItem",
                "dynamodb:Query",
                "dynamodb:Scan",
                "dynamodb:UpdateItem"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:dynamodb:*:*:table/*-stack-AWSEBWorkerCronLeaderRegistry*"
            ]
        },
        {
            "Sid": "ECSAccess",
            "Effect": "Allow",
            "Action": [
                "ecs:StartTask",
                "ecs:StopTask",
                "ecs:RegisterContainerInstance",
                "ecs:DeregisterContainerInstance",
                "ecs:DiscoverPollEndpoint",
                "ecs:Submit*",
                "ecs:Poll"
            ],
            "Resource": "*"
        }
    ]
}


## Create New Policy
Role Name: AWSElasticBeanstalkInstanceProfile

## Create New Role
Role Name: aws-elasticbeanstalk-instance-profile-role


## Create New User
User Name: RailsAssetDistributionCustomActionUser
Policy: aws-elastic-beanstalk-instance-profile  <---- check if this is needed
Policy: AWSCodePipelineCustomActionAccess
Policy: AmazonS3FullAccess

# Create IAM instance policy for Elastic Beanstalk Worker
User Name: RailsAssetDistributionCustomActionUser
Policy: AWSCodePipelineCustomActionAccess
Policy: AmazonS3FullAccess


# Create an Elastic Beanstalk App via the Console
## New Environment
* Worker Environment
## Environment Type
* Predefined Configuration = Ruby
* Environment type: Load balancing, auto scaling
## Application Version
* Source = Existing application version, "Sample Application"
* Deployment Limits = 30%
## Environment Info
* Environment name = railsassetdist-env
* Description =
## Additional Resources
* None
## Configuration Details
* Instance Type = t2.medium
* EC2 key pair = <whatever you use to log into your machien>
* email address = markmans+ebworker@amazon.com
* health check URL = /health_check
## Environment Tags
* AWS_ACCESS_KEY_ID=<ACCESS ID>
* AWS_SECRET_ACCESS_KEY=<ACCESS KEY>
* S3_ASSET_BUCKET=markmans-reinvent-demo-assets
* AWS_DEFAULT_REGION=us-east-1

## Worker Details
* Worker queue = Autogenerated queue
* HTTP path = /
* Mime type = application/json
* HTTP connections = 10
* Visibility timeout = 300
## Permissions
* Instance Profile: <create IAM role>
* Service Role: <create the default role>


# Setup CodePipeline
## Add in the custom action
$ aws codepipeline create-custom-action-type --cli-input-json file://lib/custom_action/RailsAssetDistribution.json


# Notes

## Create a Rails Application to process AWS CodePipeline Jobs
$ rails new rails-asset-distribution-custom-action -O # skips active record

## CLI
eb init --region us-east-1
--application-name CustomAction 
--environment-name RailsAssetDistCustomActionEnv 
--solution-stack XXX 
--application-name rails-asset-distribution-custom-action-for-aws-code-pipeline

$ aws elasticbeanstalk create-application --application-name "RailsAssetDistributionCustomAction" --description "AWS CodePipeline custom action for distributing Rails assets"
$ aws elasticbeanstalk create-environment \
  --application-name "RailsAssetDistributionCustomAction" \
  --description "" \
  --solution-stack-name "64bit Amazon Linux 2015.03 v2.0.0 running Ruby 2.2 (Passenger Standalone)" \
  --option-settings \
    Namespace=aws:elasticbeanstalk:application:environment,OptionName=AWS_ACCESS_KEY_ID,Value=XX \
	Namespace=aws:elasticbeanstalk:application:environment,OptionName=AWS_SECRET_KEY,Value=XX



# Simulate from rails console
region = 'us-east-1'
asset_bucket = "codepipeline-reinvent-demo-assets"
custom_action = AssetDistributorCustomAction.new(region, asset_bucket)
custom_action.poll_for_jobs
# status = custom_action.process_job
custom_action.has_new_job?

custom_action.process_job

OR


poll_results = custom_action.poll_results
job = AssetDistributorJob.new(poll_results.jobs.first)
meta_data = job.meta_data
custom_action.acknowledge_job
# build_artifact = distrubute_rails_assets_to_s3
distributor = AssetDistributor.new(region,
                                   meta_data.location.s3_location,
                                   meta_data.name,
                                   asset_bucket)
distributor.download
distributor.unzip
distributor.sync_unzipped_assets



