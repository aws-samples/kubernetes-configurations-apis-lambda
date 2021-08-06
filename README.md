*This sample, non-production-ready template describes managing EKS with Lambda functions.
(c) 2021 Amazon Web Services, Inc. or its
affiliates. All Rights Reserved. This AWS Content is provided subject to the
terms of the AWS Customer Agreement available at
http://aws.amazon.com/agreement or other written agreement between Customer
and either Amazon Web Services, Inc. or Amazon Web Services
EMEA SARL or both.*


# Simplifying Kubernetes configurations using AWS Lambda

Creating a serverless model for updating Elastic Kubernetes Clusters (EKS)

This repository enables users to call Kubernetes APIs to create and manage resources through a unified control plane. Users will interact with Kubernetes API using Python and the the config map is created by Jinja2. This provides a solution that simplifies the user experience by enabling users to manage a Kubernetes cluster without installing multiple tools on their local developer machine. Additionally, this solution will remove the complexities of additional Domain-specific language knowledge and reduce the dependencies and packages installed on the user’s local machine.

## Repository Layout:

```bash
.
├── Dockerfile
├── LICENSE
├── README.md
├── app
│   ├── app.py
│   └── templates
│       └── aws-auth.yaml.jinja
├── events
│   └── example-event.json
└── iam
    ├── lambda-role-permission.json
    └── lambda-trust-policy.json

4 directories, 8 files

```

## Requirements:

* Permission to deploy a Docker image to AWS Elastic Container Repository (ECR) and trigger Lambda functions within a given AWS environment. See sample policy below:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AccountPermissions",
            "Effect": "Allow",
            "Action": [
                "ecr:GetAuthorizationToken",
                "ecr:PutRegistryPolicy",
                "ecr:PutReplicationConfiguration",
                "lambda:ListFunctions",
                "lambda:ListEventSourceMappings",
                "lambda:GetAccountSettings",
                "lambda:ListLayers",
                "lambda:ListLayerVersions",
                "lambda:ListCodeSigningConfigs"
            ],
            "Resource": "*"
        },
        {
            "Sid": "ECRSpecific",
            "Effect": "Allow",
            "Action": [
                "ecr:PutLifecyclePolicy",
                "ecr:PutImageTagMutability",
                "ecr:GetDownloadUrlForLayer",
                "ecr:ListTagsForResource",
                "ecr:UploadLayerPart",
                "ecr:BatchDeleteImage",
                "ecr:ListImages",
                "ecr:PutImage",
                "ecr:UntagResource",
                "ecr:BatchGetImage",
                "ecr:CompleteLayerUpload",
                "ecr:DescribeImages",
                "ecr:TagResource",
                "ecr:DescribeRepositories",
                "ecr:StartLifecyclePolicyPreview",
                "ecr:InitiateLayerUpload",
                "ecr:DeleteRepositoryPolicy",
                "ecr:BatchCheckLayerAvailability",
                "ecr:ReplicateImage",
                "ecr:GetRepositoryPolicy",
                "ecr:GetLifecyclePolicy"
            ],
            "Resource": "arn:aws:ecr:*:<ACCOUNT-ID>:repository/*"
        },
        {
            "Sid": "LambdaLayerSpecific",
            "Effect": "Allow",
            "Action": [
                "lambda:GetLayerVersion",
                "lambda:GetLayerVersionPolicy",
                "lambda:GetProvisionedConcurrencyConfig",
                "lambda:DeleteLayerVersion",
                "lambda:GetEventSourceMapping",
                "lambda:ListFunctionsByCodeSigningConfig",
                "lambda:GetCodeSigningConfig"
            ],
            "Resource": [
                "arn:aws:lambda:*:<ACCOUNT-ID>:function:*:*",
                "arn:aws:lambda:*:<ACCOUNT-ID>:event-source-mapping:*",
                "arn:aws:lambda:*:<ACCOUNT-ID>:layer:*:*"
            ]
        },
        {
            "Sid": "LambdaSpecific",
            "Effect": "Allow",
            "Action": [
                "lambda:InvokeFunction",
                "lambda:ListVersionsByFunction",
                "lambda:GetFunction",
                "lambda:ListAliases",
                "lambda:PublishLayerVersion",
                "lambda:UpdateFunctionConfiguration",
                "lambda:GetFunctionConfiguration",
                "lambda:GetFunctionCodeSigningConfig",
                "lambda:UpdateFunctionCode",
                "lambda:ListFunctionEventInvokeConfigs",
                "lambda:ListProvisionedConcurrencyConfigs",
                "lambda:GetFunctionConcurrency",
                "lambda:ListTags",
                "lambda:GetFunctionEventInvokeConfig",
                "lambda:DeleteFunction",
                "lambda:PublishVersion",
                "lambda:GetAlias",
                "lambda:GetPolicy"
            ],
            "Resource": [
                "arn:aws:lambda:*:<ACCOUNT-ID>:layer:*",
                "arn:aws:lambda:*:<ACCOUNT-ID>:function:*"
            ]
        }
    ]
}
```

#### Additional Lambda Role Permissions
The Lambda role needs the following permissions:
1. Create CloudWatch Logs
2. The following permissions below

```json
{
    "Version": "2012-10-17",
    "Statement": [        
        {
            "Sid": "EKSSpecific",
            "Effect": "Allow",
            "Action": [
                "eks:DisassociateIdentityProviderConfig",
                "eks:UpdateClusterConfig",
                "eks:AssociateIdentityProviderConfig"
            ],
            "Resource": [
                "arn:aws:eks:*:<ACCOUNT-ID>:identityproviderconfig/*/*/*/*",
                "arn:aws:eks:*:<ACCOUNT-ID>:cluster/*"
            ]
        }
  ]
}
```

## Implementation Steps:

1a. Create Lambda role
```bash
aws iam create-role \
--role-name <ROLE-NAME> \
--assume-role-policy-document file://iam/lambda-trust-policy.json
```
1b. Create IAM policy
```bash
aws iam create-policy \
--policy-name <ROLE-NAME>-policy \
--policy-document file://iam/lambda-role-permission.json
```
add basic lambda execution role:
```bash
aws iam attach-role-policy \
--role-name <ROLE-NAME> \
--policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```
2. Create AWS ECR
```bash
aws ecr create-repository \
--repository-name <ECR-NAME>
```
and authorization to push Docker images to ECR:
```bash
aws ecr get-login-password \
--region <REGION> | docker login \
--username AWS \
--password-stdin <ACCOUND-ID>.dkr.ecr.<REGION>.amazonaws.com
```
3a. Create Elastic Kubernetes Cluster:
```bash 
eksctl create cluster \
--name demo-eks-cluster \
--version 1.20 \
--nodegroup-name demo-managed-node-group \
--node-type t3.medium \
--nodes 2 \
--region <REGION> \
--enable-ssm
```
3b. Add Lambda role to EKS configmap
```bash
kubectl edit -n kube-system configmap/aws-auth
```
3c. Add the following
```bash
- userarn: <LAMBDA-ROLE-ARN>
    username: admin
    groups:
    - system:masters
```

4. Create and push Docker image to repository:

```bash
docker build -t blog-example:1.0 .;
docker tag blog-example:1.0 <ACCOUNT-ID>.dkr.ecr.<REGION>.amazonaws.com/<ECR-NAME>:latest;
docker push <ECR-URI>:latest
```

5. Create Lambda container function:
```bash
aws lambda create-function \
--function-name <FUNCTION-NAME> \
--package-type Image \
--code ImageUri=<ECR-URI>:latest \
--role <LAMBDA-ROLE-ARN>
```

6. Run API command via Lambda:

```bash
aws lambda invoke \
--function-name <LAMBDA-NAME> \
--invocation-type Event \
--payload fileb://events/event.json \
response.json
```

API Layout is as such

```json
{
  "RequestType" : "Create",
  "ResourceProperties" : {
    "ClusterName": "<CLUSTER-NAME>",
    "RoleMappings": [
      {
        "arn": "<IAM-ARN-TO-ADD>",
        "username": "system:node:{{EC2PrivateDNSName}}",
        "groups": [
          "system:bootstrappers",
          "system:nodes"
        ]
      }
      <ADDITIONAL-ROLE-MAPPINGs>...
    ]
  }
}
```

# Additional Information

1. Allowed API "RequestTypes": Create, Update, Delete
2. To verify run the following command: `kubectl edit configmap -n kube-system aws-auth`

## Security:

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License:

This library is licensed under the MIT-0 License. See the LICENSE file.

