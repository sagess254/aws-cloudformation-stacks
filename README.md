# Automation with AWS CloudFormation

A hands-on AWS lab on Infrastructure as Code. I deployed a CloudFormation stack from a YAML template, then edited the template to add an Amazon S3 bucket and an Amazon EC2 instance, updated the stack, and finally deleted it so CloudFormation cleaned up every resource it had created.



---

## Table of Contents

- [Why CloudFormation?](#why-cloudformation)
- [What the Lab Covers](#what-the-lab-covers)
- [Template Anatomy](#template-anatomy)
- [Task 1: Deploy a CloudFormation Stack](#task-1-deploy-a-cloudformation-stack)
- [Task 2: Add an Amazon S3 Bucket](#task-2-add-an-amazon-s3-bucket)
- [Task 3: Add an Amazon EC2 Instance](#task-3-add-an-amazon-ec2-instance)
- [Task 4: Delete the Stack](#task-4-delete-the-stack)
- [Key Takeaways](#key-takeaways)
- [Stack Lifecycle Reference](#stack-lifecycle-reference)

---

## Why CloudFormation?

Deploying infrastructure by hand is error-prone: people skip steps, take undocumented shortcuts, and it is hard to deploy out-of-hours. CloudFormation solves this by describing infrastructure in a **template** that can be deployed consistently and repeatedly, even on an automated schedule.

## What the Lab Covers

- Deploy a stack that defines a **VPC** and a **Security Group**
- Extend the stack with an **S3 bucket** and an **EC2 instance**
- **Delete** the stack and its resources

---

## Template Anatomy

The starting template (`task1.yaml`) is written in YAML, where indentation matters (two spaces per level). CloudFormation also supports JSON.

| Section | Purpose | In this lab |
|---|---|---|
| `Parameters` | Inputs prompted for at deploy time and reused in the template | Two CIDR ranges, for the VPC and the subnet |
| `Resources` | The infrastructure to deploy | VPC, subnet, route table, security group and related networking |
| `Outputs` | Selected information exposed about the stack | The VPC's default security group |

---

## Task 1: Deploy a CloudFormation Stack

Deploys a VPC, subnet and security group from the provided template.

1. Download `task1.yaml` and read it in a plain text editor (not a word processor).
2. In the console, go to **CloudFormation → Create stack → Upload a template file** and choose `task1.yaml`.
3. Set **Stack name** to `Lab`. The CIDR parameters have defaults, so I left them unchanged.
4. Leave the stack options at their defaults and acknowledge the custom resource names on the review page.
5. Click **Create stack**.

While it runs, two tabs are worth watching:

- **Events:** a reverse-chronological log of what CloudFormation is doing, and where any errors appear.
- **Resources:** the resources being created. CloudFormation works out the order itself, for example creating the VPC before the subnet.

The stack is done when its status changes from `CREATE_IN_PROGRESS` to `CREATE_COMPLETE`.

---

## Task 2: Add an Amazon S3 Bucket

Goal: edit the template to add a bucket, then update the stack. The lab expects you to find the syntax in the *Amazon S3 Template Snippets* documentation.

Under `Resources:`, a bucket needs only a logical ID and a `Type` (no properties):

```yaml
  S3Bucket:
    Type: AWS::S3::Bucket
```

(`S3Bucket` is the logical ID. Any unique name works.)

**Update the stack**

1. Select the `Lab` stack, then click **Update**.
2. Choose **Replace current template → Upload a template file** and select the edited `task1.yaml`.
3. Click through the remaining pages and review the **change preview**.

The preview shows a single **Add** for the S3 bucket, with all other resources unchanged. This shows that CloudFormation only changes what differs between the old and new templates, so adding to an existing stack is fast.

After the status reaches `UPDATE_COMPLETE`, the bucket appears on the **Resources** tab. CloudFormation gives it a random name so it can't clash with an existing bucket.

---

## Task 3: Add an Amazon EC2 Instance

An EC2 instance depends on other resources (an AMI, a security group and a subnet), so it is more involved than the bucket.

### 3.1 Add a parameter for the AMI

In the `Parameters` section:

```yaml
  AmazonLinuxAMIID:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2
```

This reads the latest Amazon Linux 2 AMI ID from **AWS Systems Manager Parameter Store**. The same template therefore works in any region without hard-coding region-specific AMI IDs.

### 3.2 Reference other resources with `!Ref`

Resources in a template refer to each other with `!Ref`. For example, the route table references the VPC:

```yaml
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
```

### 3.3 Define the instance

Under `Resources:`, using the five properties required by the lab:

```yaml
  AppServer:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref AmazonLinuxAMIID
      InstanceType: t3.micro
      SecurityGroupIds:
        - !Ref AppSecurityGroup
      SubnetId: !Ref PublicSubnet
      Tags:
        - Key: Name
          Value: App Server
```

Notes:

- `ImageId` references the AMI parameter added in 3.1.
- `SecurityGroupIds` expects a **list**, so the reference is written as a YAML list item.
- `AppSecurityGroup` and `PublicSubnet` are defined elsewhere in the template.

### 3.4 Update the stack

Update the stack with the revised template, as in Task 2. The change preview should show the EC2 instance being added. Once the update completes, the instance appears under **Resources**, and in the EC2 console as **App Server**.

---

## Task 4: Delete the Stack

1. Select the `Lab` stack and click **Delete → Delete stack**.
2. The status shows `DELETE_IN_PROGRESS`, and after a few minutes the stack disappears.

CloudFormation deletes everything it created (the VPC and its networking, the S3 bucket and the EC2 instance), so nothing is left behind to incur cost. I verified in the VPC, S3 and EC2 consoles that the resources were gone.

> **Note:** S3 buckets must be empty before CloudFormation can delete them. In this lab the bucket was empty, but in real stacks a non-empty bucket will cause the delete to fail.

---

## Key Takeaways

- **Infrastructure as Code** gives consistent, repeatable, documented deployments.
- **Parameters** make templates reusable, and the **SSM parameter type** removes hard-coded, region-specific AMI IDs.
- **`!Ref`** links resources together, and CloudFormation works out the creation order from those references.
- **Change previews** show exactly what an update will add, modify or replace before anything is applied.
- **Updates are incremental.** Only changed resources are touched, so unchanged ones aren't redeployed.
- **Deleting a stack** cleans up all of its resources in one step, which keeps lab and test environments tidy.
- YAML is **indentation-sensitive**, so most template errors come from a misplaced indent or a missing list hyphen.

## Stack Lifecycle Reference

| Status | Meaning |
|---|---|
| `CREATE_IN_PROGRESS` / `CREATE_COMPLETE` | Stack is being created, or creation succeeded |
| `UPDATE_IN_PROGRESS` / `UPDATE_COMPLETE` | Stack is being updated, or the update succeeded |
| `DELETE_IN_PROGRESS` | Stack and its resources are being deleted |

---
