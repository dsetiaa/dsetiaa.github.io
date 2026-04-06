---
layout: post
title: "Handling Unhealthy Nodes in EKS"
categories: Infrastructure
author: Divyanshu
---

Kubernetes nodes in an EKS cluster can occasionally enter a `NotReady` state due to issues like network outages, unresponsive kubelet, or hardware problems. 
When a node becomes NotReady, its Pods may be evicted or stuck terminating, and new Pods won’t schedule there. Strategies to detect and remediate these NotReady 
nodes are thus extremely important for maintaining cluster health and application availability. In the case of ML deployments, where the nodes might require large GPUs,
and thus might be pretty expensive, timely handling of unhealthy NotReady nodes can also lead to huge cost savings in cloud bills. 

This post explores three approaches to handle NotReady nodes in an Amazon EKS cluster, especially when using Karpenter for autoscaling:
1. Using a metric-based CloudWatch alarm to send an email notification.
2. Using a metric-based alarm to trigger an AWS Lambda for automated remediation.
3. Relying on Karpenter’s Node Auto Repair feature for automated in-cluster healing.

## 1. Metric-Based Alarm and Email Notification
### How It Works
The simplest approach is to set up a CloudWatch alarm on a metric that indicates node readiness, and have it notify the dev-ops team via email (using Amazon SNS). 
AWS Container Insights provides the metric `node_status_condition_ready`, which is 1 when a node is `Ready` and 0 when `NotReady​`. 

You can set up a CloudWatch alarm to trigger when the `node_status_condition_ready` metric drops to 0 for any node — indicating it has entered a NotReady state. 
CloudWatch Metric Insights simplify monitoring across dynamic clusters, and can be used to aggregate this metric across all nodes. 
For instance, taking the minimum value across all nodes helps detect if even a single node becomes `NotReady`. 
This approach eliminates the need to manually manage alarms as nodes scale in and out. 
When the alarm fires, it can publish to an SNS topic, which then sends an email alert to your team.


### Steps:
You can follow these steps to set this system up for your cluster.

1. Create a SNS topic
```
aws sns create-topic --name Node-not-ready-alerts-topic
```
2. Get the topic ARN and save it to a shell variable
```
TOPIC_ARN="arn:aws:sns:us-east-1:123456789012:MyTopic"
```

3. Subscribe to the topic with an email address
```
aws sns subscribe \
  --topic-arn "$TOPIC_ARN" \
  --protocol email \
  --notification-endpoint "you@example.com"
```

Once done. You will get an email about topic subscription confirmation
- Check the inbox for “AWS Notification - Subscription Confirmation”.
- Click the Confirm subscription link.

4. (Optional) Send a test notification
```
aws sns publish \
 --topic-arn "$TOPIC_ARN" \
 --subject "SNS test message" \
 --message "Hello from AWS CLI!"
```
You should receive an email in couple of minutes.

5. Create an alarm for the `node_status_condition_ready` metric
```
aws cloudwatch put-metric-alarm \
  --alarm-name "node-not-ready-alarm" \
  --namespace "ContainerInsights" \
  --metric-name node_status_condition_ready \
  --dimensions Name=ClusterName,Value=your-cluster \
  --statistic Minimum \                   
  --period 60 \                           
  --evaluation-periods 5 \                
  --threshold 1 \                         
  --comparison-operator LessThanThreshold \
  --alarm-description "Fires when any node in your-cluster is not Ready" \
  --alarm-actions "$TOPIC_ARN"
```
### Architecture Diagram

<img src="/assets/images/handling-unhealthy-nodes-in-eks/alarm-email-notification.png" />

### Pros and Cons

#### Pros
- **Simple to implement:** Uses native AWS monitoring (CloudWatch, SNS) with no custom code.
- **No automation risks:** It won’t accidentally terminate the wrong node – you decide what to do.
- **Low cost and maintenance:** Email alarms are straightforward and low-cost.
- **Works in any cluster:** Doesn’t depend on cluster autoscaler behavior or Karpenter versions.

#### Cons
- **Manual intervention required:** Engineers must manually remediate the node, resulting in slower recovery.
- **Downtime potential:** The node remains NotReady until someone reacts, which could impact workloads.
- **Alert fatigue:** In large clusters or flapping conditions, repeated emails can overwhelm on-call staff.
- **May miss sequential failures:** If two nodes fail close together, the alarm may only trigger on the first one. 
Since CloudWatch alarms require a cooldown period (based on the evaluation window and datapoints), the second failure might occur 
before the alarm resets, leading to no alert being sent. This is especially common when using longer periods to reduce false positives for high-capacity nodes.


## 2. Metric-Based Alarm Triggering a Lambda
### How It Works

For a more automated solution, you can have the CloudWatch alarm invoke an AWS Lambda function to repair the node. 
The monitoring part is the same – use the `node_status_condition_ready` metric and an alarm through CloudWatch. 
However, instead of (or in addition to) sending an email, the alarm’s action targets a Lambda via the SNS trigger. 
The Lambda function contains logic to delete the NotReady node. This in turn triggers replacement capacity. 
For example, the Lambda can call the Kubernetes API to cordon and delete the node. Deleting the Node object evicts 
any remaining pods and frees up the name so that the cluster autoscaler can provision a new node if needed. 

### Lambda Remediation Pseudocode 
A simple Python pseudocode for the Lambda’s logic might look like:
```
def handler(event, context):
    # Identify the affected nodes (By scanning cluster state)
    affected_nodes = get_notready_nodes()

    if not affected_nodes:
        return "No action needed"
    
    # Delete the nodes from the cluster.
    for node in affected_nodes
        delete_node(affected_node)

    return f"Deleted nodes {affected_nodes}"
```

To enable the function to do this, it would require appropriate permissions and to establish the SNS topic as a trigger. All of that can be done using the following
CloudFormation template. Once you've created and uploaded the Lambda image in a private ECR repository, and have created the SNS topic, all you need to do is pass
the image URI and the topic ARN as parameters into the template, along with the cluster name.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: "Resources to configure AWS Lambda with access to the cluster"
Parameters:
  ClusterName:
    Type: String
    Description: "Cluster Name"
    Default: tensorkube
  EksAccessLambdaRepoURI:
    Type: String
    Description: "URI of the EKS access Lambda repository"
  EksAccessLambdaFunctionImageVersion:
    Type: String
    Description: "Version of the EKS access Lambda function image"
  NodeAlertSNSTopicArn:
    Type: String
    Description: ARN of the SNS topic that sends NotReady node alerts
Resources:
  LambdaExecutionRole:
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
      Path: "/"
      Policies:
        - PolicyName: describe-tensorkube-cluster
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - eks:DescribeCluster
                Resource: !Sub "arn:aws:eks:${AWS::Region}:${AWS::AccountId}:cluster/${ClusterName}"
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource: "*"
              - Effect: Allow
                Action:
                  - ecr:GetDownloadUrlForLayer
                  - ecr:BatchGetImage
                  - ecr:BatchCheckLayerAvailability
                  - ecr:GetAuthorizationToken
                Resource: "*"
        - PolicyName: RoleTrustPolicyUpdateAccess
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - iam:UpdateAssumeRolePolicy
                  - iam:GetRole
                Resource: !Sub "arn:aws:iam::${AWS::AccountId}:role/${ClusterName}-*"
      Tags:
        - Key: CreatedBy
          Value: Tensorfuse
        - Key: ClusterName
          Value: !Ref ClusterName


  ClusterAccessEntry:
    Type: AWS::EKS::AccessEntry
    Properties:
      ClusterName: !Ref ClusterName
      PrincipalArn: !GetAtt LambdaExecutionRole.Arn
      AccessPolicies:
        - AccessScope:
            Type: cluster
          PolicyArn: "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"
      Tags:
        - Key: CreatedBy
          Value: Tensorfuse
        - Key: ClusterName
          Value: !Ref ClusterName
    DependsOn: LambdaExecutionRole

  ClusterAccessFunction:
      Type: AWS::Lambda::Function
      Properties:
        PackageType: Image
        Code:
          ImageUri: !Sub "${EksAccessLambdaRepoURI}:${EksAccessLambdaFunctionImageVersion}"
        Role: !GetAtt LambdaExecutionRole.Arn
        Timeout: 900
        MemorySize: 1024
        EphemeralStorage:
          Size: 2048
        Environment:
          Variables:
            CLUSTER_NAME: !Ref ClusterName
        Description: "Lambda function to access the cluster"
        Tags:
          - Key: CreatedBy
            Value: Tensorfuse
          - Key: ClusterName
            Value: !Ref ClusterName
      DependsOn:
        - ClusterAccessEntry

  LambdaSNSSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      Protocol: lambda
      TopicArn: !Ref NodeAlertSNSTopicArn
      Endpoint: !GetAtt ClusterAccessFunction.Arn

  LambdaInvokePermission:
    Type: AWS::Lambda::Permission
    Properties:
      Action: lambda:InvokeFunction
      FunctionName: !Ref ClusterAccessFunction
      Principal: sns.amazonaws.com
      SourceArn: !Ref NodeAlertSNSTopicArn

Outputs:
  LambdaFunction:
    Value: !Ref ClusterAccessFunction
    Description: "Lambda function to access the cluster"
  LambdaFunctionArn:
    Value: !GetAtt ClusterAccessFunction.Arn
    Description: "ARN of the Lambda function"
```

### Architecture Diagram

<img src="/assets/images/handling-unhealthy-nodes-in-eks/alarm-lambda-remediation.png" />


### Pros and Cons

#### Pros
- **Fast, automated recovery:** Reduces the time a `NotReady` node affects the cluster by programmatically replacing it by removing any dependency on human intervention.	
- **Customizable logic:** The Lambda can include organization-specific rules (e.g. drain only if certain conditions, notify only if others) beyond simple node deletion.	
- **Quicker scaling clusters:** Especially in large clusters, automated handling prevents `NotReady` nodes from piling up.	


#### Cons
- **Increased complexity:** Requires writing, testing, and maintaining the Lambda code (including cluster API access and AWS permissions).
- **Lambda won't retrigger for rapid successive failures:** If two nodes become NotReady within the same alarm window, the Lambda is only invoked for the first one. 
The second failure may be missed entirely if it occurs while the alarm is still in the ALARM state. 
This tradeoff arises from the need for longer evaluation periods to prevent false positives, especially with large nodes that take time to register as Ready.

## 3. Karpenter Node Repair Feature
### How It Works
Karpenter (the AWS-backed cluster autoscaler for Kubernetes) includes a Node Auto Repair capability designed to detect and replace unhealthy nodes within 
the cluster itself. This feature, introduced in `v1.1.0`, allows Karpenter’s controller to monitor node health conditions and automatically terminate and 
recreate nodes that remain unhealthy beyond a certain time window​
​
In essence, Karpenter acts as the “self-healing” agent, so you don’t need external Lambdas or custom alarms for node health. 
Under the hood, Karpenter watches Kubernetes Node conditions. It needs a node monitoring agent to be installed to observe the node statuses. 
So if a node’s Ready status turns to False (NotReady) or Unknown and stays that way for more than 30 minutes, Karpenter will initiate node repair​.

“Repair” in this context means Karpenter will forcefully terminate the node’s instance and remove the Node object, bypassing normal graceful termination. 
This is done to ensure the unhealthy node is quickly taken out of service and that scheduling can happen on new nodes​

When Karpenter deletes the Node (and the associated NodeClaim/instance), any Pods that were on it will be rescheduled according to their disruption policies. 
If those Pods require a node (and the cluster is under-provisioned), Karpenter’s provisioning logic will launch a new replacement instance to satisfy the 
pending Pods. In many cases, Karpenter can even anticipate this by seeing the node is unschedulable and starting a replacement before the old one is fully gone, 
ensuring a smoother transition of workloads.

To use this feature, you need to be running a Karpenter version `>=v1.1.0` and enable the NodeRepair feature gate. This typically involves setting the Helm chart value on the Karpenter controller (`--settings.feature-gates.nodeRepair=true`). 
You also need deploy the AWS Node Monitoring Agent (or Node Problem Detector) if you want Karpenter to react to non-Ready conditions​

Once enabled, no further action is needed – Karpenter continuously checks node health and will log and take action when a node is deemed unhealthy. 
To prevent cascading failures, Karpenter will halt auto-repairs if more than 20% of the nodes in the cluster (or in the specific provisioning group) are unhealthy.

### Architecture Diagram

<img src="/assets/images/handling-unhealthy-nodes-in-eks/karpenter-node-repair.png" />


### Pros and Cons

#### Pros

- **Integrated solution:** No external scripts or cloud services are needed – Karpenter handles detection and replacement within the Kubernetes control plane​
- **Holistic health monitoring:** Can react to various node health signals (Ready status, kernel, network, storage issues)
- **Minimal operational toil:** Once set up, there’s no need to wake up on-call for common node failures – the cluster self-heals. Karpenter ensures new nodes come in to replace bad ones automatically.
- **Safety mechanisms:** Karpenter won’t evict too many nodes at once (20% rule) to protect cluster stability​. It also won’t interfere with managed node group auto-repair (if you use mixed provisioning).	

#### Cons
- **Alpha feature caveats:** As of now, Node Auto Repair is a relatively new feature and there may be corner cases. Default toleration periods (e.g. 30 minutes for NotReady) are reasonable, but you can't adjust Karpenter’s settings to fit your needs.
- **Potentially forceful:** Karpenter uses force termination after a timeout. There is a small risk of data loss for pods that don’t gracefully shut down in time, since it skips some drain grace period for speed​
- **Limited immediate feedback:** No built-in notification. If a node was replaced, you might only see it via cluster events or Karpenter logs. So you may still want an alarm on node replacements or unhealthy-node events for awareness.

​
## Summary Comparison Table


| Aspect                   | Email Notification                                | Lambda Deletion                                          | Karpenter Node Repair                                      |
|--------------------------|----------------------------------------------------|-----------------------------------------------------------|-------------------------------------------------------------|
| **Mechanism**            | CloudWatch alarm notifies humans via SNS email.   | CloudWatch alarm triggers Lambda to programmatically fix the node. | Karpenter controller detects and replaces unhealthy nodes in-cluster. |
| **Automation**           | Notification only (manual fix).                 | Automatic remediation (node removal).                 | Automatic remediation (node removal and replacement).     |
| **Response Time**        | Human-dependent (minutes to hours).               | Near real-time (Lambda runs within seconds of alarm).     | Semi real-time (reacts after ~10–30 mins of unhealthy status. Thresholds are unconfigurable right now). |
| **Implementation Effort**| Very low (configure alarm & SNS topic).           | Medium (write & deploy Lambda + alarm).                  | Medium (install/upgrade karpenter + enable feature gate + install health monitoring agent).       |
| **Scope of Health Signals** | Basic Ready status only.                         | Basic Ready status only.                       | Wide node conditions (e.g., Ready, GPU health, network).   |
| **Alerting Visibility**  | Direct alert to on-call (via email/SNS).          | Can alert/log action via SNS or logs.                    | Relies on cluster logs/events (add alerts manually if needed). |


## Conclusion

Unhealthy Kubernetes nodes are inevitable in any sizable EKS cluster. So it is important to have robust safeguards in place to quickly deal with them whenever they arise.

A basic alarm with email notification is easy to set up and ensures the right people know about the problem – it’s a good starting point for awareness, though it relies on human intervention. 
Moving up, an alarm that triggers a Lambda can automatically remediate the issue, which drastically cuts resolution time and can keep your cluster stable with minimal human effort at the cost of more setup and careful testing of the automation. 
Finally, Karpenter’s Node Auto Repair offers an elegant in-cluster solution – once you enable it, your cluster itself (via Karpenter) will actively maintain node health by replacing bad nodes on the fly​. 
This feature aligns with cloud-native self-healing principles and can simplify operations, especially as it matures into a stable offering. 

In practice, you might combine approaches. For example, even with Karpenter auto-repair on, you may still keep a CloudWatch alarm to notify you when nodes are being replaced, so you’re aware of underlying infrastructure issues. 
Always consider the reliability requirements of your workloads: for critical clusters, an automated repair plus an alert (so you know it happened) can provide both high availability and transparency. 
