# EKS with CloudFormation

EKS with CloudFormation is a lab I wanted to build to quickly set up and unset. There are some steps to prepare pipelines, IAM Roles and whist not the most recommended way of doing, it helped me to learn more about CloudFormation and EKS.

## EKS templates for different roles and stacks
I used different stacks by adapting **eksctl** templates. I wanted to learn inside them and adapt them to run with variables exported and imported between stacks, so in the future I could write an automation to set them up, or a Portfolio in Service Catalog, but not just there yet. Focus is setting up EKS we can create and destroy quickly while getting more experienced on CloudFormation.
1) Network
2) EKS Cluster
3) IAM Role
We used a different IAM role template from AWS documentation which extracts and outputs in CloudFormation Output section the OIDC from the cluster. It requires a Lambda function and the code can be found on \> xxxxx
4) Node Group
## Pipelines in CodePipeline to deploy stacks
I wanted to have more experience on pipelines, so this project started with a simple 3 stage, Source, Create ChangeSet and Execute ChangeSet steps, but be trying more complex stage validations such as linter (CFN-Lint).

### IAM Role for the pipelines
For this lab I have created an IAM role for CloudFormation use in the pipelines which have admin rights and more IAM capabilities while troubleshooting the initial setup. Later, those need to be changed to less privileges.` The IAM Role iam::288693765212:role/Custom\_PipelineDeployStageIaC` has administrator privileges for now. 
Also since this role is used by CloudFormation, it will be the admin/owner of the EKS cluster. So this role needs to trust your IAM user and AWS Account to assume it. This is how the JSON policy looks like. Without this permission, the role cannot be assumed and we would get the error:

```bash
An error occurred (AccessDenied) when calling the AssumeRole operation: User: arn:aws:iam::288693765212:user/admin is not authorized to perform: sts:AssumeRole on resource: arn:aws:iam::288693765212:role/Custom_PipelineDeployStageIaC
```

![](Screenshot%202021-07-03%20at%2015.36.32.png)

```bash
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudformation.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    },
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::288693765212:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Also, your IAM user needs to be able to assume the CloudFormation role. Here is how it is done as a inline policy:
```bash
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::288693765212:role/Custom_PipelineDeployStageIaC"
        }
    ]
}
```
And, by the way, this could be a policy applied to a group of IAM users. This allows groups of users to have access to groups of clusters and/or services easier, and to revoke access by group membership. Going further, it is possible to use conditionals to validate dates or other criteria such as Tags putting a more granular control.


##  Local kubectl setup
Kubectl can be configured quickly using the command:

```bash
aws eks update-kubeconfig --region us-east-1  --name EKS-01-Cluster-eks-cloudformation --role-arn arn:aws:iam::288693765212:role/Custom\_PipelineDeployStageIaC 
```

Make sure you have AWS CLI set up first with your IAM user credentials. If you are using multiple AWS profiles, just add the “—profile” parameter to the command.

Test with the `kubectl get svc` command.

```bash
rodrigo@iMac cfn-lint-main % kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   172.20.0.1   <none>        443/TCP   6h30m
rodrigo@iMac cfn-lint-main % 
```

