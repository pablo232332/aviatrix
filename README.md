# Deploying Aviatrix on AWS using CloudFormation

This repository contains AWS CloudFormation templates designed to facilitate the deployment of Aviatrix platform components within an AWS environment.

## Templates Included

| Template File                  | Description                                                                                             | Notes                                                                    |
| :----------------------------- | :------------------------------------------------------------------------------------------------------ | :----------------------------------------------------------------------- |
| `aviatrix_deployment.yml`| Deploys the Aviatrix Controller (BYOL model) EC2 instance, required IAM roles, policies, and security group. | Base component. Can be "original" (with IAM conditions) or "simplified". |
| `aviatrix-template.yml` | Deploys the backend infrastructure for Aviatrix's Terraform-based CloudFormation Custom Resource Types.   | **Optional:** Only needed if using specific `Aviatrix::*::*` custom types. |

## Prerequisites

* An active AWS Account.
* Sufficient IAM permissions to create resources (EC2, IAM Roles/Policies, S3, Lambda, Security Groups, CloudFormation Stacks).
* A valid Aviatrix License (for the BYOL model deployed by `aviatrix-controller-byol.yaml`).
* Access to the AWS Management Console or a configured AWS CLI.

## Aviatrix Controller Deployment (`aviatrix_deployment.yml`)

This template is the first step to getting the Aviatrix platform running.

### Purpose

Launches the Controller EC2 instance and sets up essential IAM components (`aviatrix-role-ec2`, `aviatrix-role-app`, related policies) and a security group for access.

### Key Parameters

| Parameter                     | Description                                                                 | Type                  | Default / Allowed Values                              | Notes                                                               |
| :---------------------------- | :-------------------------------------------------------------------------- | :-------------------- | :---------------------------------------------------- | :------------------------------------------------------------------ |
| `VPCParam`                    | VPC ID for Controller deployment.                                           | `AWS::EC2::VPC::Id`   | -                                                     |                                                                     |
| `SubnetParam`                 | **Public** Subnet ID within the selected VPC.                               | `AWS::EC2::Subnet::Id`| -                                                     | Must have a route to an Internet Gateway.                           |
| `AllowedHttpsIngressIpParam`  | IP address or CIDR range allowed for HTTPS (443) access to the Controller.  | `String`              | e.g., `1.2.3.4/32`                                    | **Crucial for security - change the default!** |
| `InstanceTypeParam`           | EC2 instance size for the Controller.                                       | `String`              | `t3.large` (Default), See template for allowed list |                                                                     |
| `IAMRoleParam` *(Original)* | Specify if IAM roles should be created (`New`) or already exist (`aviatrix-role-ec2`). | `String`              | `New`, `aviatrix-role-ec2`                            | Only present in the original, more complex template version.        |

### Deployment Instructions

1.  Navigate to the AWS CloudFormation service in the AWS Console.
2.  Click "Create stack" -> "With new resources (standard)".
3.  Choose "Upload a template file" and select `aviatrix-controller-byol.yaml`.
4.  Fill in the required parameters (VPC, Subnet, Allowed IP, etc.). If using the original template and roles already exist, select the appropriate option for `IAMRoleParam`.
5.  Review the options and confirm stack creation.
    * Alternatively, use the AWS CLI: `aws cloudformation create-stack --stack-name YourControllerStackName --template-body file://aviatrix-controller-byol.yaml --parameters ParameterKey=...,ParameterValue=... --capabilities CAPABILITY_IAM`

### Important Notes & Troubleshooting

* **Public Subnet Required:** Ensure the selected subnet has a route to an Internet Gateway.
* **BYOL Model:** This template deploys the "Bring Your Own License" version. You need a valid Aviatrix license.
* **"AlreadyExists" IAM Errors:** If you use the simplified template (or the original with `New`) and encounter errors stating that roles (`aviatrix-role-ec2`, `aviatrix-role-app`) or policies (`aviatrix-app-policy`, `aviatrix-assume-role-policy`) already exist, it means they were created in a previous attempt. You have two main options:
    1.  **Manually Delete:** Delete the existing IAM resources (roles, policies, the `aviatrix-role-ec2` instance profile) from the IAM console **if you are certain they are safe to remove**. Then, retry the deployment.
    2.  **Use Original Template:** Use the original template version and set the `IAMRoleParam` parameter to `aviatrix-role-ec2` to instruct CloudFormation to skip creating these resources.
* **Security Group Description Characters:** The `GroupDescription` property must contain only **ASCII characters**. Edit the template if you've used non-ASCII characters (like accents).

### Stack Outputs

| Output Name                   | Description                              | Value Source                             |
| :---------------------------- | :--------------------------------------- | :--------------------------------------- |
| `AviatrixControllerEIP`       | Public IP address of the Controller.     | `!Ref AviatrixEIP`                       |
| `AviatrixControllerPrivateIP` | Private IP address of the Controller.    | `!GetAtt AviatrixController.PrivateIp`   |

---

## Terraform-CloudFormation Handler Deployment (`aviatrix-template.yml`)

Deploy this template **only** if you need to use Aviatrix's Terraform-based custom resource types within your CloudFormation templates.

### Purpose

Installs the backend Lambda function, IAM roles, and S3 bucket required to handle `Aviatrix::*::*` custom resource types (powered by Terraform) within CloudFormation.

### When to Use

**Only** if you plan to use these specific Aviatrix custom resource types (e.g., `Aviatrix::TGW::Transit`) in other CloudFormation templates. It is **not required** for basic Controller operation or management via the Controller UI/Terraform Provider directly.

### Key Parameters

| Parameter         | Description                                                                                                | Type     | Default / Allowed Values | Notes                                                           |
| :---------------- | :--------------------------------------------------------------------------------------------------------- | :------- | :----------------------- | :-------------------------------------------------------------- |
| `AllowAWSActions` | If `true`, grants broad (`*.*`) IAM permissions to the handler Lambda. Use with caution.                     | `String` | `false` (Default), `true`| Only set to `true` if required by the Terraform modules used. |
| `S3Bucket`        | Optional: Custom S3 bucket containing the Lambda function code.                                            | `String` | `''` (Default)           | Overrides the default Aviatrix-provided bucket.                 |
| `S3Key`           | Optional: Custom S3 key for the Lambda function code ZIP package within the specified/default bucket.        | `String` | `''` (Default -> `app.zip`)| Overrides the default key (`app.zip`).                          |

### Deployment Instructions

The process is similar to deploying the Controller:
1.  Navigate to the AWS CloudFormation service.
2.  Create a new stack using the `aviatrix-cfn-tf-handler.yaml` template file.
3.  Configure the parameters (especially review `AllowAWSActions`).
4.  Create the stack.
    * Alternatively, use the AWS CLI: `aws cloudformation create-stack --stack-name YourHandlerStackName --template-body file://`aviatrix-template.yml --parameters ParameterKey=...,ParameterValue=... --capabilities CAPABILITY_IAM`

### Stack Outputs

| Output Name        | Description                                                                     | Value Source                 |
| :----------------- | :------------------------------------------------------------------------------ | :--------------------------- |
| `ExecutionRoleARN` | ARN of the IAM role used by CloudFormation when handling these custom types. | `!GetAtt ExecutionRole.Arn`  |

---

## Deployment Flow & Relationship

1.  **Deploy Controller:** Always deploy `aviatrix-controller-byol.yaml` first to set up the core Controller.
2.  **Deploy Handler (Optional):** If you intend to use Aviatrix Terraform-based custom types in CloudFormation, deploy `aviatrix-cfn-tf-handler.yaml`.
3.  **Use Custom Types:** Create other CloudFormation stacks that utilize the `Aviatrix::*::*` custom resource types. You will likely need to configure these types or the stack itself to use the `ExecutionRoleARN` output from the handler stack.

### Component Diagram
