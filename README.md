In the repo, we are creating a solution on AWS Cloud using native services that will utilise AWS CloudWatch agent to monitor the disk utilization of EC2 instances across AWS Accounts and regions. 
We will utilize AWS CloudWatch alarm to watch the `disk_used_percent` metrics and if the disk usage crosses the threshold of 85%, the solution will initiate an automation using AWS Systems Manager to exteend the size of the EBS volume and send alerts ut
For the deployment of the solution, we are using AWS CloudFormation StackSets to deploy the solution across all AWS accounts that are part of the AWS Organization, this can be configured at a single AWS account as well. 



