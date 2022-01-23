# Valheim AWS ECS Spot (CloudFormation)

This CloudFormation template can be used to deploy a Valheim dedicated server to Amazon Web Services (AWS) using Elastic Container Service (ECS) using an EC2 Spot Instance in minutes.  Usage of an Spot Instance can provide significant cost savings over a on-demand instance, and the world file is saved to an Elastic File System (EFS) for persistance.  An option is provide to stop the Server/Instance, with the world file preserved, for even greater cost savings when not required.

## Prerequisites

1. An AWS Account and basic understanding of Amazon Web Services [AWS](https://aws.amazon.com/), and specifically [CloudFormation](https://aws.amazon.com/cloudformation/).
2. If you want to migrate an existing World, knowledge of Linux and Docker administration (generally no more than what would be required to just use the [mbround18/valheim](https://hub.docker.com/r/mbround18/valheim) Docker image).

## Overview

The solution builds upon the [mbround18/valheim](https://hub.docker.com/r/mbround18/valheim) Docker image, so generously curated by [mbround18](https://github.com/mbround18) and [contributors](https://github.com/mbround18/valheim-docker/graphs/contributors) - Thank You!  Thanks also goes to [m-chandler](https://github.com/m-chandler) for [factorio-spot-pricing](https://github.com/m-chandler/factorio-spot-pricing) (and [contributors](https://github.com/m-chandler/factorio-spot-pricing/graphs/contributors)) upon which this solution and documentation is based.

The CloudFormation template launches an _ephemeral_ [EC2 Spot Instance](https://aws.amazon.com/ec2/spot) which joins itself to an [Elastic Container Service (ECS)](https://aws.amazon.com/ecs/) Cluster. Within this ECS Cluster, an ECS Service is configured to run the above Valheim Docker image. The ephemeral instance does not store the world lcaolly, this is stored on persistent [Elastic File System (EFS)](https://aws.amazon.com/efs/).

**IMPORTANT NOTE:** While EC2 Spot Instance can provide upto a 90% discount compared to On-Demand EC2 prices, if the current Spot Price exceeds your configured `SpotPrice` maximum limit ([along with a few other termination reasons](https://aws.amazon.com/premiumsupport/knowledge-center/ec2-spot-instance-termination-reasons/)) it will be terminated with a 2 minute warning. With this CloudFormation template the world is saved to EFS for persistance, and a new server will automatically be launched if/when applicable capacity is available using an [Auto Scaling Group (ASG)](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html).  However is you prefer a more traditional VPS style hosting environment, I would suggest [hosting on Amazon Lightsail](https://aws.amazon.com/getting-started/hands-on/valheim-on-aws/).

## Getting Started

1. Logon to AWS and navigate to [Cloudformation](https://us-west-2.console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/create/template?stackName=valheim).
2. Ensure you have selected your preferred AWS Region at the top right.  Typically this will be the nearest to your location, but Spot Instances can also vary by region.
3. Under specify template select Upload and provide the [`cf.yaml`](https://raw.githubusercontent.com/apeabody/Valheim-AWS-ECS-Spot/main/cf.yaml) from this repository.
4. Click 'Next' to proceed through the CloudFormation deployment and provide parameters on the following page. At a minimum you will need a `ServerName` and `WorldName` (these can't be the same) and server `Password`.  For Webhooks and other parameters please see the [mbround18/valheim](https://hub.docker.com/r/mbround18/valheim) Docker image documentation.

## Next Steps

It may take a while for your Valheim server to start, especially if you are using an older/small (and less expensive) instance type such as the suggested m3.medium.  Once CloudFormation reports the stack status as `CREATE_COMPLETE` you can check the server's Public IP address in the [EC2 dashboard in the AWS console](https://console.aws.amazon.com/ec2/v2/home?#Instances:sort=instanceId) however the Valheim server process might still take a while to finish starting.  For a more automated method the `WEBHOOKURL` and `WEBHOOKINCLUDEPUBLICIP` parameters can be used for the server to post notifications in a Discord channel, including the server's public IP.  This can be especially beneficial when using Spot Instances as capacity can fluctuate and the instance automatically relaunch in a different availability zone.

## FAQ

**How do I login to the server directly?**

Access directly to the server shouldn't be generally required, but support for [EC2 Instance Connect](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Connect-using-EC2-Instance-Connect.html) in built-in to the CloudFormation template.  To enable, enter the [EC2 Instance Connect IP range](https://ip-ranges.amazonaws.com/ip-ranges.json) for your chosen region in the `EC2ICRANGE` parameter and follow the standard EC2 Instance Connect method.  If you have experience with Linux and ECS and/or Docker this method can also be used to download existing world saves to the EFS.

**What if my server is terminated due to my Spot Request being outbid?** 

Your world is stored on persistent EFS, so if the server had time to cleanly terminate and save you won't lose any progress, and the world is otherwise periodically saved.  If all goes well the ASG should launch a new instance shortly, perhaps in a different Availability Zone and with a new IP address.  If you desired instance/price is no longer available long term you might need to adjust your CloudFormation stack's `InstanceType` or `SpotPrice` parameters.

**My server keeps getting terminated and/or I don't want to use a Spot Instance?** 

While you could update your CloudFormation stack's `SpotPrice` parameter to an empty value for [EC2 On-Demand pricing](https://aws.amazon.com/ec2/pricing/on-demand/), I would suggest looking at [hosting on Amazon Lightsail](https://aws.amazon.com/getting-started/hands-on/valheim-on-aws/) for a more traditional VPS environment.

**How do I change my instance type?** 

Update your CloudFormation stack's `InstanceType` parameter.

**How do I change my spot price limit?** 

Update your CloudFormation stack's `SpotPrice` parameter. 

**I'm done for the night / week / month / year. How do I turn off my Valehim server?** 

Update your CloudFormation stack's `ServerState` parameter from "Running" to "Stopped".

**I'm done with Valheim, how do I delete this server?** 

Delete the CloudFormation stack, everything except for the persistent EFS will be terminated.  If you are absolutely sure you no longer want your World(s), you can also delete the EFS.  In normal circumstances the EFS cost should be minimal as a lifecycle policy is included to move files to Infrequent Access after 7 days which at the time of writing is [$0.025 per GB-Month](https://aws.amazon.com/efs/pricing/).

**I'm running the "latest" version, and a new version has just been released. How do I update?** 

You can `Force new deployment` of the service via ECS.

## Expected Costs

The two primary cost generators are:

* **EC2** - If you're using spot pricing (and the default m3.medium instance the template), this is likely to be under $0.01 per hour which is about $7.50 per month.  However you can use the CloudFormation stack's `SpotPrice` parameter to enforce a chosen maximum price per hour.
* **EFS** - While it varies by region, Standard EFS cost is typically less than [$0.50 per GB-Month](https://aws.amazon.com/efs/pricing/).

There are other AWS charges, such as data egress, but these are generally minimal.  **HOWEVER**, if you are at all concerned about cost I would strongly recommend using [AWS Budgets](https://aws.amazon.com/aws-cost-management/aws-budgets/) which can notify on both actual and forecasted cost.

## Help / Support

This has been tested in the US-WEST-2 (Oregon) AWS region (verify your AWS region of choice includes m3.medium @ https://aws.amazon.com/ec2/spot/pricing and/or change the instance type/size as needed).  If assistance with the AWS Cloudformation/Deployment is required please create an [Issue](https://github.com/apeabody/Valheim-AWS-ECS-Spot/issues).  You may also find [mbround18/valheim]https://github.com/mbround18/valheim-docker useful for the underlying Docker image.

### Stack gets stuck on CREATE_IN_PROGRESS

Your chosen instance size might be not available in your region.  I would recommend checking the Spot Request > Pricing History in the AWS EC2 Console and update the CloudFormation stack's `InstanceType` parameter. 

## Thanks

Thanks goes out to [m-chandler](https://github.com/m-chandler) for [factorio-spot-pricing](https://github.com/m-chandler/factorio-spot-pricing) (and [contributors](https://github.com/m-chandler/factorio-spot-pricing/graphs/contributors)) upon which this solution is based.

Thanks also goes out to [mbround18/valheim](https://hub.docker.com/r/mbround18/valheim) (and [contributors](https://github.com/mbround18/valheim-docker/graphs/contributors)) for maintaining the Valheim Docker images.