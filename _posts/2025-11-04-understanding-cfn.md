---
layout: post
categories: [Infrastructure, AWS, IaC]
author: Divyanshu
title: "CloudFormation won't fix your life but it can fix your infra"
description: A look into how efficiently managing your cloud infrastructure will definitely make your life easier, even though it may not fix it
---



<div class="note">
This blog focuses on AWS specific resources and services
</div>

If you’ve ever worked with cloud infrastructure, one thing will have become abundantly clear to you: managing cloud resources is a pain. Period. It’s extremely easy to lose track of all the resources you’ve provisioned for a project and how their state might have changed since you provisioned them. So it becomes very obvious that having a written record of all these resources would be extremely convenient. It would also be so great if any changes you made to your written records are automatically reflected in your cloud resources.  

This is where Infrastructure as Code(IaC) tools come in. They do exactly that\! They allow you to provision and manage infrastructure through code instead of manual processes.

### What is CloudFormation?

CloudFormation is (one of) AWS’s IaC service(s). It allows you to specify, in a yaml/json format, all the resources you want AWS to spin up, along with all the properties you want the resources to have. Your wish is AWS’s command (given that you have the money and quotas and also follow proper wishing format without screwing up indentation).  
You can update this yaml and AWS will make the corresponding changes to the resources you want.

### Doesn’t something better exist?

Yes and No. (obviously :P)  

As I mentioned before, CloudFormation (CFN) is ONE of AWS’s IaC services. There is also, Cloud Development Kit (CDK). YAMLs can get pretty verbose and hard to read and manage. And if you’re not familiar with reading and writing yaml files, starting off with CFN can be difficult. CDK provides an abstracted layer over CFN and its yaml files. It allows you to define resources in programming languages that you are familiar with [(Python, Java, C\#, Go, TS, JS)](https://docs.aws.amazon.com/cdk/v2/guide/languages.html), thus also enabling you to leverage IDE tools for debugging and code completion. It also provides `Constructs` as abstractions over cloud resources. Constructs can be individual resources (low-level constructs), and also groups of common resource patterns, for e.g. a Fargate Service with a load-balancer (high-level constructs). You can also create your own constructs, thus enabling you to create modules specific to your workflow, and allowing for code reusability. Using a programming language also enables you to use loops, which can’t be done in CFN, and also makes conditionals more intuitive.  
Now, technically you can implement loops in CFN with the help of macros, which I talk about later in this blog, but CFN does not allow you to do that out of the box.  

All that being said, CDK is an ABSTRACTION over CFN, and definitionally, abstracts out details and fine grained control. And at the end of the day, CDK also generates CFN templates, and you need to be able to understand CFN if you ever need to debug something.  

However, the primary reason we chose CFN over CDK, was that CFN enabled us to [import our cloud resources and map them](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/resource-import-existing-stack.html) to the resources defined in our YAML files. CDK did not allow that. We would have had to create new resources and delete the older ones to switch to CDK, which we did not want to do.  

<div class=”note”>
There is an <code>import</code> function in CDK, but that only enables you to refer to existing resources while creating new ones, not map them to a resource you define in your code. The generated CFN templates(yaml) do not have a resource definition for the cloud resource you “imported” in your code.
</div>

It may still make sense for you to use CDK. We had a strict requirement to migrate all our resources to an IaC service **in one go** and the way our other systems were created did not allow for replacing the resources with newer ones and then deleting the older resources. If that does not hold true for you or if you can slowly migrate to IaC, or maybe handle some downtime, it may be well worth it to use CDK. I have also not worked much with CDK, so this is the extent till which I know about it and you should definitely try out both services before you decide what to use.  

If you decide to use CFN, welcome to the club\! If you chose CDK, I would still recommend understanding the basics of CFN because, like I said, under CDK’s hood, it’s all still CFN.


<img src="/assets/images/understanding-cfn/cfn-diagram.png" />



### The basics of CloudFormation

Every tool comes with its own terminology, and CloudFormation is no exception. Thankfully, with CFN, it’s all pretty intuitive. Here are the most important ones:

- #### Templates and Stacks  
  A CFN `Template` is the yaml or json file that specifies all the resources and their configurations. It serves as blueprints for creating your AWS Resources. A `Stack` is the set of corresponding resources that are created in the cloud. It is the instantiation of a Template. You create, update, and delete a collection of resources by creating, updating, and deleting stacks. All stack operations are atomic and failures invoke rollback (except delete which enters a `Delete Failed` state)

	Templates must follow a format specified by AWS. It looks something like this:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Simple stack with one S3 bucket
Resources:
  Bucket:  
    Type: AWS::S3::Bucket
```

- #### Resources  
  A `Resource` is, well, the AWS resource you want to provision and configure. You specify resources in the, aptly named, `Resources` section of the CFN template. Each resource must have a Type attribute, which defines the kind of AWS resource it is. The Type attribute has the format `AWS::ServiceName::ResourceType`. For example, the Type attribute for an Amazon S3 bucket is `AWS::S3::Bucket`. You can also define dependencies between resources using the `DependsOn` attribute.  
  
  Example:  
```yaml
  Resources:  
    Role:  
      Type: AWS::IAM::Role  
      Properties:  
        AssumeRolePolicyDocument:  
          Version: '2012-10-17'  
          Statement:  
            - Effect: Allow  
              Principal:  
                Service: lambda.amazonaws.com  
              Action: sts:AssumeRole
```

	

<div class="note">
You can specify a CloudFormation stack as a resource as well! These are called nested stacks and are very useful in logically grouping different resources together in the different nested stacks. For example, we had a base stack whose resources were just various nested stacks; one for our EKS cluster, one for all the addons, one for OIDC provider for the cluster, one for resources related to Lambda functions we want to configure etc. This also helped keep each individual stack less verbose and helped us stay way below the resource-per-template limit of 500.
</div>

- #### Resource Properties  
  Resource properties help define configuration details for the specific resource type. They are strongly typed and can be literal strings, lists of strings, Booleans, dynamic references, parameter references, pseudo references, or the value returned by a function. Some properties are required, while others may have default values and are optional.   
  
  Here is the [official example](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/resources-section-structure.html#resources-section-resource-properties) on how you can declare different property value types:  

```    
  Properties:  
    String: A string value   
    Number: 123  
    LiteralList:  
      - first-value  
      - second-value  
    Boolean: true
```

  The official AWS documentation for the properties of each supported resource type and their details is very comprehensive and absolutely amazing. You can check it out [here](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-template-resource-type-ref.html%20)

- #### Parameters  
  The Parameters section of a CFN Template is an optional but incredibly useful section, allowing you to customise your stacks without altering the template itself, by providing input values during stack creation and updation. For instance, you might use a parameter to vary the instance type of a resource, depending on the environment settings that vary between deployments.  
  
  By using parameters in your templates, you can build reusable and flexible templates that can be tailored to specific scenarios.   
  
  Each parameter has a name and a type, and can have additional settings such as default value and allowed values. The parameter type determines the kind of input value the parameter can accept. For example, `Number` for numeric values and `String` for text inputs. You can check out more information about their syntax [here](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html).  
  
  Example:  
```yaml
  Parameters:  
    InstanceType:  
      Type: String  
      Default: t3.micro  
      AllowedValues:  
        - t3.micro  
        - t3.small  
        - t3.medium  
```
    
  And you would reference a parameter in a resource like so:  
```yaml
  Resources:  
    MyEC2Instance:  
      Type: 'AWS::EC2::Instance'  
      Properties:  
        InstanceType: !Ref InstanceType
```
    
  Parameters are incredibly useful in specifying property values of stack resources. However, there may be settings that are region dependent or are somewhat complex, where more logic might be required. And you can put (some of) logic in the template itself using `Conditions` and `Mappings`\! [Conditions](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/conditions-section-structure.html) are fairly straightforward and basically allow you to add `if else` statements in your templates. I never worked with Mappings as much so you can read more about them [here](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/mappings-section-structure.html).   
- #### Outputs  
  **Outputs** expose key stack results, such as resource ARNs, URLs, or IDs. Outputs can be consumed manually or programmatically through the AWS CLI, SDKs, or as part of CI/CD workflows.  
    
  Outputs help you configure which values are returned when viewing a stack’s properties. They can help retrieve and use important information like resource ARNs, URLs, IDs etc. about the resources created by the template.   
  
  Outputs provide useful information such as resource identifiers or URLs, which can be leveraged for operational purposes or for integration with other stacks.  
```yaml
  Outputs:  
    LambdaArn:  
      Description: ARN of the deployed Lambda function  
      Value: !GetAtt MyLambda.Arn  
```
    
- #### Exports  
  Exports are a special type of outputs that enable you to reference the outputs of one CloudFormation stack in other stacks. This allows for modular templates and cross-stack dependencies. You can reference an export from a stack in another stack using the `Fn::ImportValue` [CFN function](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference.html). It’s important to note that export names must be unique across all stacks.  
  
  Example:   

**StackA.yaml**
```yaml
  #Export from this stack
  Outputs:  
    VPCId:  
      Value: !Ref VPC  
      Export:  
        Name: SharedVPCId  
```   

**StackB.yaml**
```yaml
  #Import into this stack
  Resources:  
    AppSubnet:  
      Type: AWS::EC2::Subnet  
      Properties:  
        VpcId: !ImportValue SharedVPCId  
        CidrBlock: 10.0.2.0/24  
```
---

There are other template sections like Conditions, Mappings, and Rules and you can read about them in more detail in the [official documentation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-anatomy.html)

Here is an example of a full template:
```yaml
AWSTemplateFormatVersion: '2010-09-09'  
Description: A CloudFormation template to create an EC2 instance with a specified instance type.

Parameters:  
  InstanceType:  
    Type: String  
    Description: The EC2 instance type.  
    Default: t3.micro  
    AllowedValues:  
      - t3.micro  
      - t3.small  
      - t3.medium  
    ConstraintDescription: Must be a valid t3 instance type.

  LatestAmiId:  
    Type: 'AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>'  
    Default: '/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2'  
    Description: The latest Amazon Linux 2 AMI ID.

  KeyName:  
    Type: AWS::EC2::KeyPair::KeyName  
    Description: Name of an existing EC2 KeyPair to enable SSH access to the instance.  
    ConstraintDescription: Must be the name of an existing EC2 KeyPair.

Resources:  
  MyEC2Instance:  
    Type: 'AWS::EC2::Instance'  
    Properties:  
      InstanceType: !Ref InstanceType  
      ImageId: !Ref LatestAmiId  
      KeyName: !Ref KeyName  
      Tags:  
        - Key: Name  
          Value: !Sub 'EC2 Instance created by CloudFormation (${AWS::StackName})'

Outputs:  
  InstanceId:  
    Description: The Instance ID of the created EC2 instance.  
    Value: !Ref MyEC2Instance  
    Export: true  
    
  PublicDnsName:  
    Description: Public DNS name of the EC2 instance.  
    Value: !GetAtt MyEC2Instance.PublicDnsName

  PublicIp:  
    Description: Public IP address of the EC2 instance.  
    Value: !GetAtt MyEC2Instance.PublicIp
```


### Asking the Template to modify itself

Sometimes, you need to dynamically modify your CloudFormation templates, like changing the number of resources on the basis of a parameter, or tag each resource in the template with specific tags, implementing dynamic conditions etc. And CFN, being declarative in nature, isn’t particularly suited for that.  

This is where CloudFormation Macros come in. Macros allow you to perform custom processing on your CloudFormation templates, from simple actions like find-and-replace operations to extensive transformations of entire templates.  

Here is more concrete example of what we needed macros for:  
CFN templates are YAML/JSON files and thus are made up of key-value pairs. In CFN, you can use Parameters or Functions to modify the “Values” of those key-value pairs, but this is not possible for the keys. And we needed to dynamically modify the keys as well. This is where we used a CFN macro to modify the template itself. More specifically, we were creating an IAM Role, whose Trust Policy needed to contain a trust relationship with our EKS cluster’s OIDC provider; The OIDC Issuer needs to be a part of the `StringEquals` key. The trust relationship looks something like this:

```yaml
Effect: Allow  
Principal:  
Federated: "arn:aws:iam::<AWS AccountId>:oidc-provider/<OIDC Issuer>"  
Action: sts:AssumeRoleWithWebIdentity  
Condition:  
    StringEquals:  
        "<OIDC Issuer>:aud": sts.amazonaws.com  
        "<OIDC Issuer>:sub": "system:serviceaccount:k8s-ns:k8s-sa"  
```

To implement this in CFN, we used the official PyPlate macro, that evaluates python code in the template
```yaml
Resources:  
  RoleIAMRole:  
    Type: AWS::IAM::Role  
    Properties:  
      RoleName: pyplate-eg-role  
      AssumeRolePolicyDocument:  
        Version: '2012-10-17'  
        Statement:  
          - Effect: Allow  
            Principal:  
              Federated: !Sub 'arn:${AWS::Partition}:iam::${AWS::AccountId}:oidc-provider/${OIDCIssuer}'  
            Action: "sts:AssumeRoleWithWebIdentity"  
            Condition:  
              StringEquals: |  
                #!PyPlate  
                output = {}  
                oidc_issuer = params["OIDCIssuer"]  
                namespace = params["Namespace"]  
                service_account_name = params["ServiceAccountName"]  
                output = {  
                    f"{oidc_issuer}:aud": "sts.amazonaws.com",  
                    f"{oidc_issuer}:sub": f"system:serviceaccount:{namespace}:{service_account_name}"  
                }
```

Macros thus allow us to extend the capabilities of CloudFormation beyond just its declarative nature. But they should also be used sparingly; transformations complicate template validation and debugging. However, for controlled dynamic generation, they significantly reduce template duplication. There is an excellent blog by the AWS team on CFN Macros that you can (and absolutely should) check out [here](https://aws.amazon.com/blogs/mt/deep-dive-on-aws-cloudformation-macros-to-transform-your-templates/), and an official repo with some examples [here.](https://github.com/aws-cloudformation/aws-cloudformation-templates/tree/main/CloudFormation/MacrosExamples) 

### What CFN won’t solve, Lambda will

The thing that makes CloudFormation so attractive is the ability to provision, configure, and manage all your related infrastructure in one place. This however breaks down in real scenarios where not all resources to be provisioned are AWS resources. You often need to configure and manage non-AWS infrastructure. These may be third-party services, or in our case, Kubernetes resources. Managing infrastructure often also requires making non-resource AWS API requests like initialising databases with data during creation, dynamically looking up AMI IDs during stack creation, or emptying buckets before deletion, etc.  

To solve this, AWS allows you to specify [CustomResources](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html) in your AWS CloudFormation templates. They provide a way for you to write custom provisioning logic into your templates and have CloudFormation run it anytime you create, update (if you changed the custom resource), or delete a stack.  

And you can associate a [Lambda](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources-lambda.html) function with a custom resource, which is invoked whenever the custom resource is created, updated, or deleted. CloudFormation calls a Lambda API to invoke the function and to pass all the request data (such as the request type and resource properties) to the function. The power and customizability of Lambda functions in combination with CloudFormation enable you to easily manage all your infrastructure in one place.  

Here is an example of how we used Custom Resources in our stacks:   
```yaml
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
        - PolicyName: EmptyS3BucketsPolicy  
          PolicyDocument:  
            Version: '2012-10-17'  
            Statement:  
            - Effect: Allow  
              Action:  
              - s3:ListBucket  
              - s3:GetObject  
              - s3:PutObject  
              - s3:DeleteObject  
              Resource: !Sub "arn:aws:s3:::${ClusterName}-*"

  AWSAccessFunction:  
      Type: AWS::Lambda::Function  
      Properties:  
        PackageType: Image  
        Code:  
          ImageUri: !Sub "${AWSAccessLambdaRepoURI}:${AWSAccessLambdaFunctionImageVersion}"  
        Role: !GetAtt LambdaExecutionRole.Arn  
```

This was invoked like this:

```yaml
Resources:  
  EnvBuildBucket:  
    Type: AWS::S3::Bucket  
    Properties:  
      BucketName: !Ref BuildBucketName  
      AccessControl: Private

  EmptyBuildBucketAction:  
    Type: Custom::InvokeLambdaFunction  
    Properties:  
      Region: !Sub "${AWS::Region}"  
      ClusterName: !Ref ClusterName  
      Operation: "empty-s3-bucket"  
      Parameters:  
        BucketName: !Ref BuildBucketName  
      AWSAccountId: !Sub "${AWS::AccountId}"  
      ServiceToken: !Ref AWSLambdaFunctionArn  
    DependsOn: EnvBuildBucket  
```

The code for empty-bucket operation was in the lambda function image and looked something like:

```python
import boto3  
import botocore.exceptions  
import cfnresponse  
from enum import Enum

class Operation(Enum):  
    EMPTY_S3_BUCKET = 'empty-s3-bucket'

def empty_s3_bucket(bucket_name: str):  
    s3 = boto3.client('s3')  
    paginator = s3.get_paginator('list_objects_v2')

    for page in paginator.paginate(Bucket=bucket_name):  
        if 'Contents' in page:  
            for obj in page['Contents']:  
                s3.delete_object(Bucket=bucket_name, Key=obj['Key'])  
    print(f"Deleted objects from bucket {bucket_name}")  
    return True

def handler(event, context):  
    command = event['ResourceProperties'].get('Operation')  
    parameters = event['ResourceProperties'].get('Parameters')

    if event['RequestType'] == 'Create':  
        pass  
    elif event['RequestType'] == 'Delete':  
 if operation == Operation.EMPTY_S3_BUCKET.value:  
        bucket_name = parameters.get('BucketName')  
        if not bucket_name:  
            raise Exception("BucketName is a required parameter")  
        empty_s3_bucket(bucket_name)  
        return cfnresponse.send(event, context, cfnresponse.SUCCESS, {'Event': 'Create', 'Reason': 'Operation successful'})  
```

You can modify this lambda code however you like and thus automate any required tasks to efficiently manage your infrastructure. In fact, the macros we created in the previous section also use AWS Lambda functions to transform CloudFormation templates\!

### What CFN and Lambda won’t solve, CodeBuild will

Now in Tensorfuse, we not only managed our own infrastructure, but also provisioned and managed our customers’ AWS infra (and optimised it for AI/ML workloads). The problem is that the aforementioned Lambdas we were using required custom images, and you can’t create Lambda functions using public ECR images. So we needed a way to distribute these Lambda functions.   
Enter AWS CodeBuild.  

[CodeBuild](https://docs.aws.amazon.com/codebuild/latest/userguide/welcome.html) is AWS’s managed build service. You can use it to compile your application code, run unit tests, **build docker images**, and other artifacts to automate your CI/CD workflows. These operations are resource intensive and can exceed Lambda’s runtime and memory limits.  
Since you can build and push docker images, with CodeBuild, you can absolutely pull images from a public repo and push them to private ECR repositories. We came across this [blog](https://medium.com/@yikaihu121/how-to-deploy-aws-lambda-with-an-image-from-ecr-public-gallery-using-cdk-cloudformation-86cc3ddf6530) that guided us on how to implement that, and were so grateful that we didn’t have to bang our heads and figure another thing out for a project that had already taken way too long.

---

Now I’m gonna stop here, but there is obviously way more to CloudFormation. Heck there are things that I wanted to/should cover but this blog is already so long.   
And there are definitely going to be so many things that I don’t even know about. Know that with CloudFormation, AWS’s documentation is your best friend. It is very comprehensive, and honestly pretty easy to go through.  
This blog was my attempt at an introduction to CloudFormation and a list of all the major problems we faced while migrating to it. If this helps you avoid even half the headaches we faced, that’s a success\! (If not, welp… sucks, I guess)

Okay that’s it\! Bye Bye. Sayonara. Good night.

---

Further reading:

- [Deletion policy](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-attribute-deletionpolicy.html): Helps configure behaviour (delete, retain, backup) of critical resources like DBs and S3 buckets during stack deletion  
- [Stack sets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.html): Help you manage stacks across regions and accounts.  
- [Stack Termination Protection](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-protect-stacks.html): Prevent accidental deletion of stacks  
- [Change sets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets.html): View how updates to stacks affect resources. ([Or do them blind](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-direct.html))  
- [Update behaviours](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-update-behaviors.html): Understand how stack and resource updates affect resources and their availability

---

Resources:

- Best Friend: [https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html)  
- [https://medium.com/@yikaihu121/how-to-deploy-aws-lambda-with-an-image-from-ecr-public-gallery-using-cdk-cloudformation-86cc3ddf6530](https://medium.com/@yikaihu121/how-to-deploy-aws-lambda-with-an-image-from-ecr-public-gallery-using-cdk-cloudformation-86cc3ddf6530)  
- [https://aws.amazon.com/blogs/aws/cloudformation-macros/](https://aws.amazon.com/blogs/aws/cloudformation-macros/)  
- [https://github.com/aws-cloudformation/aws-cloudformation-macros/blob/master/PyPlate/python.yaml](https://github.com/aws-cloudformation/aws-cloudformation-macros/blob/master/PyPlate/python.yaml)  
- [https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html)  
- [https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/resource-import-existing-stack.html](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/resource-import-existing-stack.html)  
- [https://docs.aws.amazon.com/cdk/v2/guide/languages.html](https://docs.aws.amazon.com/cdk/v2/guide/languages.html)   
- [https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-protect-stacks.html](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-protect-stacks.html)  
- [https://repost.aws/knowledge-center/cloudformation-accidental-updates](https://repost.aws/knowledge-center/cloudformation-accidental-updates)  
- [https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.html](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.html)  
- [https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-attribute-deletionpolicy.html](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-attribute-deletionpolicy.html)  
- [https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-direct.html](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-direct.html)  
- [https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets.html](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets.html)  
- [https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-update-behaviors.html](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-update-behaviors.html)  
  