# Parse Server Cloudformation Examples

This repository consists of two example architectures for deploying the [Parse Server](https://github.com/parse-community/parse-server) project on AWS.

## Quick intro to Cloudformation (Infrastructure as Code)
[AWS Cloudformation](https://aws.amazon.com/cloudformation/) provide a template-based way of creating infrastructure and managing dependencies between resources during the creation process. With Cloudformation, you can maintain your infrastructure just like application source code.

## Architecture overview

There are two example architectures in this repository: development and production. Both architectures share the following characteristics: 
* run a fork of the parse-server-example project: [chrisckchang/parse-server-example](https://github.com/chrisckchang/parse-server-example). The forked code is saved as a zip on AWS S3 [chrisckchang-parse.zip](https://s3.amazonaws.com/changy/chrisckchang-parse.zip).
* use [AWS Elastic Beanstalk](https://aws.amazon.com/elasticbeanstalk/), a Platform-as-a-Service, to deploy the application.
* expects a remote MongoDB database. You can use [mLab](https://mlab.com) for a free MongoDB database.

It is recommended to run a fork of the parse-server-example project so that you can update the project dependencies to your preference. For example, the upstream repository uses a minor version (4.11) of the Express library that is several versions behind the current release as of this writing (4.16).

### Development example

The development example consists of two Cloudformation templates that together create a deployment experience similar to the Heroku deployment flow.

The `parse-development-example.yaml` creates the following AWS resources:

* VPC
* Internet Gateway
* 1 public subnet in the `us-east-1a` AZ
* Elastic Beanstalk application and environment running as a "SingleInstance" environment type.
* 1 t2.micro EC2 instance ($8.47/month) in public subnet with a public IP
    * EC2 Security group with HTTP and SSH access from any IP

A couple development characteristics to note:

The Elastic Beanstalk environment runs as a "SingleInstance" environment type. Only one EC2 instance is created, which minimizes costs. However, if any failure occurs to either the application code or the underlying EC2 instance, the application will no longer be available.

The EC2 instance that runs your application is publicly accessible to the open Internet, which is not secure.

Review the production example below to see how to address these shortcomings.

#### AWS CodePipeline for automatic deploys from GitHub

The `parse-development-codepipeline-example.yaml` template creates an AWS CodePipeline pipeline that mimics the Heroku commit-to-deploy behavior. The pipeline will periodically poll the source GitHub repository for changes and deploy the new code to your Elastic Beanstalk environment if it finds any new commits. The pipeline only costs $1/month.

## Production example

**Disclaimer**: This is only an example architecture reference that does not go in-depth on many production requirements such as: monitoring & alerting, logging, testing, or security. If you intend on running Parse Server in production on AWS, first consult other resources such as the [AWS Well-Architected Framework](https://d1.awsstatic.com/whitepapers/architecture/AWS_Well-Architected_Framework.pdf) or the [Production Readiness Review](https://www.youtube.com/watch?v=QKpB0lNL5rQ) video.

The production example consists of the `parse-development-example.yaml` template, which creates the following AWS resources:

* VPC
* Internet Gateway
* 2 public subnets: 1 in the `us-east-1a` AZ, 1 in `us-east-1b`. Both contain a NAT Gateway to enable instances in the private subnet to connect to the Internet and AWS services (e.g. Elastic Beanstalk)
* 2 private subnets: 1 in the `us-east-1a` AZ, 1 in `us-east-1b`. Both contain the EC2 instances that power your application. The EC2 instances do not have a public IP address and are not reachable from the Internet.
* A [bastion host](https://docs.aws.amazon.com/quickstart/latest/linux-bastion/architecture.html) in the `us-east-1b` public subnet to allow an administrator to SSH into the EC2 instances in private subnets.
* Elastic Beanstalk application & environment running in a "LoadBalanced" environment type. The Load Balancer runs in both of the public subnets. The environment also has autoscaling enabled to ensure that there are always two instances running.

A couple production characteristics to note:

The development example places the EC2 instance that powers the application in a public subnet that is accessible from any IP. In this example, the EC2 instances that power your application do not have a public IP and are in private subnets that are not accessible to the Internet.

The production example is highly available and fault tolerant. We have configured:
* an autoscaling group to ensure that there are always two EC2 instances running.
* 2 private subnets in separate Availability Zones that host our EC2 instances
* 2 public subnets in separate Availability Zones that host our Load Balancer nodes
* 2 NAT Gateways in separate Availability Zones, each of which is highly available within an AZ

## How to deploy

To use these examples: 

1. Clone this repo
2. Log in to the [AWS console for Cloudformation in us-east-1] (https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks?filter=active).
3. Click "Create Stack"
4 Choose a template -> Upload a template to Amazon S3.
5. Upload the template you'd like to use
    * If you're using the development example, deploy the `parse-development-example.yaml` template first. The CodePipeline template requires Elastic Beanstalk Application and Elastic Beanstalk Environment names.

Note: The Stack creation process might take a few minutes!

Both the development and production templates will require the following parameters:

* APP_ID (you can use any String value)
* MASTER_KEY (you can use any String value)
* DATABASE_URI ([how to find your mLab URI](http://docs.mlab.com/connecting/#connect-string)) 
* EC2KeyName (the [AWS SSH keyPair](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html) name to allow SSH access to the EC2 host)

The CodePipeline template uses a GitHub repository as a source and uses OAuth tokens. Generate a GitHub token and ensure you enable the following scopes:

* `admin:repo_hook`, which is used to detect when you have committed and pushed changes to the repository
* `repo`, which is used to read and pull artifacts from public and private repositories into a pipeline

## Thanks

Thanks to the Stelligent team for having a great CodePipeline tutorial to follow: https://github.com/stelligent/dromedary.


