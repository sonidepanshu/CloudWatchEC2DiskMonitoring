In this repo, we are creating a solution on AWS Cloud using native AWS services that will utilize the AWS CloudWatch Agent to monitor disk utilization of EC2 instances across AWS Accounts and Regions.
We will use the AWS CloudWatch Dashboard to monitor the disk usage of all instances within a region in one place, and CloudWatch Alarms with Amazon SNS to send notifications whenever the defined threshold is crossed.

![highlevel](https://github.com/sonidepanshu/CloudWatchEC2DiskMonitoring/blob/main/flowaatHigherLevel.png)

At a higher level,
**Step 1:** The CloudWatch Agent running inside the EC2 instances sends data to the CloudWatch service, where a custom CloudWatch Alarm monitors the `disk_used_percent` metric and checks if it exceeds the defined utilization threshold.
**Step 2:** When an EC2 instance’s EBS volume crosses the defined utilization threshold, the CloudWatch Alarm notifies the SNS topic, which then sends an email alert to the configured email address.

### How to deploy the solution

1. **Login to Systems Manager > Fleet Manager** and configure the Default Host Management Configuration

   * Systems Manager Fleet Manager is a free service that simplifies fleet management, including running automations.
   * [Configuration guide](https://docs.aws.amazon.com/systems-manager/latest/userguide/fleet-manager-default-host-management-configuration.html)

2. **Install the CloudWatch Agent**, which is a requirement for this solution, and ensure that the IAM Role attached to the EC2 instances has the `CloudWatchAgentServerPolicy` managed policy.

   * You can follow either manual or automated steps to install the CloudWatch Agent on your EC2 fleet.
   * [Installation guide](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/install-CloudWatch-Agent-on-EC2-Instance.html)

3. **Create an SNS topic** and a **subscription** to that topic. This SNS topic will send notifications through available protocols including email, SMS, HTTP/HTTPS, Lambda, and others.

   * [Creating an Amazon SNS topic](https://docs.aws.amazon.com/sns/latest/dg/sns-create-topic.html)

4. **Set up a CloudWatch Alarm** for the `disk_used_percent` (Linux) or `Available bytes` (Windows) metrics by defining your alarm thresholds.

   * In the CloudWatch > Alarms console, click **“Create alarm”**.
   * Select the **“CWAgent”** namespace and filter the metrics by filesystem type (`fstype`).
     For Linux, select `xfs` to view metrics for all attached EBS volumes.
   * Choose the `disk_used_percent` metric for all EC2 instances and click **“Select metrics”**.
   * In the **“Specify metric and conditions”** section, add a static condition and define the threshold as “Greater than”.
     Example: to receive an alert at 85%, set the threshold to 85.
   * [Metrics collected by the CloudWatch Agent](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/metrics-collected-by-CloudWatch-agent.html#CloudWatch-agent-metrics-definitions-calculations)
   * [Create a CloudWatch Alarm for an instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-cloudwatch-createalarm.html)
   * In the alarm configuration, specify the SNS topic created in Step 3 to send notifications when the threshold is reached.

5. **Create a CloudWatch Dashboard**

   * In the CloudWatch > Dashboard console, click **“Create dashboard”**, select **Alarms** as the data type, and add the alarm created in Step 4.
   * The dashboard will display disk utilization across all EC2 instances in the region.

---

### CloudFormation Template

To automate the deployment of the SNS topic, SNS subscription, CloudWatch Alarm, and CloudWatch Dashboard:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: >
  CloudFormation Template to create SNS Topic, Email Subscription, 
  CloudWatch Alarm, and Dashboard for monitoring EC2 Disk Usage.

Parameters:
  EmailAddress:
    Type: String
    Description: Email address to subscribe to SNS topic for disk usage alerts.

  InstanceId:
    Type: String
    Description: EC2 Instance ID to monitor disk usage (e.g., i-0abcd1234efgh5678).

Resources:
  DiskUsageAlertsTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: DiskUsageAlertsTopic
      DisplayName: Disk Usage Alerts Topic

  DiskUsageEmailSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      Protocol: email
      Endpoint: !Ref EmailAddress
      TopicArn: !Ref DiskUsageAlertsTopic

  RootDiskUsedPercentAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmDescription: "Triggers when root volume disk usage exceeds 85%"
      Namespace: CWAgent
      MetricName: disk_used_percent
      Statistic: Average
      Period: 300
      EvaluationPeriods: 1
      Threshold: 85
      ComparisonOperator: GreaterThanThreshold
      AlarmActions:
        - !Ref DiskUsageAlertsTopic
      Dimensions:
        - Name: InstanceId
          Value: !Ref InstanceId
        - Name: path
          Value: "/"
        - Name: fstype
          Value: "ext4"

  DiskUsageDashboard:
    Type: AWS::CloudWatch::Dashboard
    Properties:
      DashboardName: DiskUsageMonitoringDashboard
      DashboardBody: !Sub |
        {
          "widgets": [
            {
              "type": "metric",
              "x": 0,
              "y": 0,
              "width": 12,
              "height": 6,
              "properties": {
                "metrics": [
                  [ "CWAgent", "disk_used_percent", "InstanceId", "${InstanceId}", "path", "/", "fstype", "ext4" ]
                ],
                "period": 300,
                "stat": "Average",
                "region": "${AWS::Region}",
                "title": "Root Disk Usage (%)"
              }
            },
            {
              "type": "text",
              "x": 0,
              "y": 7,
              "width": 12,
              "height": 3,
              "properties": {
                "markdown": "### 📊 EC2 Disk Usage Alert Dashboard\nThis dashboard shows disk usage metrics for the EC2 instance `${InstanceId}`.\nAlerts are sent when disk usage > 85%."
              }
            }
          ]
        }

Outputs:
  SNSTopicArn:
    Description: ARN of the SNS Topic for disk usage alerts.
    Value: !Ref DiskUsageAlertsTopic

  DashboardName:
    Description: Name of the CloudWatch Dashboard.
    Value: !Ref DiskUsageDashboard

  AlarmName:
    Description: Name of the CloudWatch Alarm.
    Value: !Ref RootDiskUsedPercentAlarm
```


