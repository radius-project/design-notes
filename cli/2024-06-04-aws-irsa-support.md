# Radius AWS Workload Identity Support

* **Status**: In Review
* **Author**: Nithya Subramanian (@nithyatsu)

## Overview

A software workload such as a container-based application, service or script needs an identity to authenticate, access, and communicate with services that are distributed across different platforms and/or cloud providers. Radius uses AWS IAM (Identity and Access Management) credentials to deploy and access AWS resources today:
 
```
rad credential register aws --access-key-id <access-key-id> --secret-access-key <secret-access-key>
 ```
 
These IAM credentials, including the access key should be rotated regularly to reduce the chance of unauthorized access. A more secure option than using these credentials is to use IRSA.
 
IRSA (IAM Roles for Service Accounts) is used for authenticating applications running within Kubernetes pods. When we use IRSA, we associate an IAM role directly with a Kubernetes service account, allowing the pods to assume that role. IRSA does not require us to configure access key and  secret. It relies identity tokens (OIDC tokens) to authenticate with AWS services. These tokens have a default expiration (configurable). However, they automatically refresh when needed, transparent to the user.

The goal of the scenario is to enable infrastructure operators to configure IRSA support for the AWS provider in Radius to deploy and manage AWS resources.

## Terms and definitions


## Objectives

> Issue Reference: https://github.com/radius-project/radius/issues/7618 

### Goals

* Radius users can configure AWS provider and enable Radius to use IRSA for authentication and deployment of application.
* IRSA can be configured via interactive experience.
* IRSA can be configured manually.
* Radius users can deploy and manage AWS resources without having to manage AWS access key id and secret

### Non-goals

* Azure Managed Identity support
* Azure Workload Identity support
* Ability of radified applications to be able to use IRSA for authenticating AWS operations.

### User scenarios

As a user, I should be able to configure IRSA with Radius for managing AWS resources.

## Design

### High Level Design for IRSA

During installation, user can enable IRSA for AWS by by choosing "IRSA" option while configuring AWS provider and providing the Role ARN. Then, service-accounts will be annotated with role arn, similar to `amazonaws.com/role-arn: arn:aws:iam::123456789012:role/rad-role`. A trust relation between the annotated service-accounts in the specified namespace with the specified OIDC provider is established this way.  

[Amazon EKS Pod Identity Webhook](https://github.com/aws/amazon-eks-pod-identity-webhook#amazon-eks-pod-identity-webhook) installed in cluster can inject neccessary configurations into the pods associated with these service-accounts for IRSA. 

At this point, we can use aws-sdk to work with IRSA to deploy and manage AWS resources. Specifically, AWS assume-role helps handle multi-tenancy. At a high level, the flow looks as below:

1. use  `rad init` to configure radius with role-arn associated with IRSA (`rad-role-arn`). This is a dummy role with no polcicies.
2. use `rad env update` to configure role-arn associated with the specific environment (`rad env update qa --aws-account-id <aws-account-id> --aws-region <aws-region> --aws-role-arn <qa-role-arn>`). This is the role to be used when user deploys from `qa` env. Should be configured with appropriate policies.
3. take steps to allow  `rad-role-arn` and `qa-role-arn` trust each other.
4. during rad deploy, radius assumes the role that is configured in the provider. Thus, if the user in `qa` env, policies/ credentials associated with `qa-role-arn` will be used.
5. because of step 4, multi-tenancy is made possible. Radius can use the irsa setup to deploy to multiple roles/accounts. 

IRSA works the same way on EKS as well as non-EKS clusters.  

### Alternatives considered

#### EKS Pod Identity 

EKS Pod Identity was introduced in 2022 as a simplified approach for applications running on EKS to retrieve credentials. It does not need setting up an OIDC identity provider and achieves this simplicity by using a new EKS service principal and APIs. 

EKS Pod Identity makes the configuration simpler, but works only on EKS workloads. 


### Architecture Diagram


### Detailed Design

#### Proposed Option

Radius should support AWS IRSA.

#### Prerequisite

1. [Setup OIDC provider] (https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html) should be complete for the cluster.
2. [Amazon EKS Pod Identity Webhook](https://github.com/aws/amazon-eks-pod-identity-webhook#amazon-eks-pod-identity-webhook) (to be setup in cluster by user as part of IRSA setup ref: [AWS IRSA documentation](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) will inject neccessary configurations into the pods associated with these service-accounts for IRSA.  
3. `rad-role` with no policies attached to it should be created. 

#### Radius support

**installation**

User can enable IRSA for AWS by by choosing "IRSA" option while configuring AWS provider and providing a base RoleARN (`arn:aws:iam::123456789012:role/rad-role`) which has no policies attached to it. The purpose of this RoleARN is to just enable the IRSA setup for Radius. As a result, the service-accounts will be annotated with role arn, similar to `amazonaws.com/role-arn: arn:aws:iam::123456789012:role/rad-role`. radius pods associated with the service-account will have environment variables  AWS_ROLE_ARN and AWS_WEB_IDENTITY_TOKEN_FILE. Also, the iam-token is mounted as a secret. 


```
    Containers:
      ucp:
        Container ID:   docker://faa5af929f91e053b04b023e05c4fa6363e87688df265443d9f377258fc3fd04
        Image:          ghcr.io/radius-project/ucpd:latest
        Image ID:       docker-pullable://ghcr.io/radius-project/ucpd@sha256:991ed703a3260a19535690242cef4d7e4d076038b016443d9106f2d76d57b0da
        Ports:          9443/TCP, 9090/TCP
        Host Ports:     0/TCP, 0/TCP
        :
        Environment:
          UCP_CONFIG:                   /etc/config/ucp-config.yaml
          BASE_PATH:                    /apis/api.ucp.dev/v1alpha3
          TLS_CERT_DIR:                 /var/tls/cert
          PORT:                         9443
          AWS_ROLE_ARN:                 arn:aws:iam::817312594854:role/rad-role
          AWS_WEB_IDENTITY_TOKEN_FILE:  /var/run/secrets/eks.amazonaws.com/serviceaccount/token
        Mounts:
          /etc/config from config-volume (rw)
          /var/run/secrets/eks.amazonaws.com/serviceaccount from aws-iam-token (ro)
          /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-d22ld (ro)
          /var/tls/cert from cert (ro)
```

At this point, a trust relation between the annotated service-accounts in the specified namespace with the specified OIDC provider to be established. 

The trust relationship for the rad-role will look like

```
  {
      "Version": "2012-10-17",
      "Statement": [
          {
              "Effect": "Allow",
              "Principal": {
                  "Federated": "arn:aws:iam::817312594854:oidc-provider/oidc.eks.us-west-2.amazonaws.com/id/67DDAC18D8C44CEDCF1C9719A8E9B866"
              },
              "Action": "sts:AssumeRoleWithWebIdentity",
              "Condition": {
                  "StringEquals": {
                      "oidc.eks.us-west-2.amazonaws.com/id/67DDAC18D8C44CEDCF1C9719A8E9B866:aud": "sts.amazonaws.com",
                      "oidc.eks.us-west-2.amazonaws.com/id/67DDAC18D8C44CEDCF1C9719A8E9B866:sub": "system:serviceaccount:radius-system:ucp"
                  }
              }
          }
      ]
  }
```

Since AWS IRSA works by annotating the service-account, enabling IRSA requires a re-install.

**post installation**

Once the installation is complete, the user can use rad env update to store relevant information for deploying resources to accounts and region associated with specific radius environment.

```
rad env update qa --aws-account-id <aws-account-id> --aws-region <aws-region> --aws-role-arn <qa-role-arn>
```

`qa-role-arn` is the role-arn of role which has the right policies associated with it. 

In addition to running `rad env update`, the user has to add a trust entry similar to below 

1. for role `qa-role` with `qa-role-arn` arn , add a trusted entity entry similar to below. 

```
{
    "Version": "2012-10-17",
    "Statement": [
        **{
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::817312594854:role/rad-role"
            },
            "Action": "sts:AssumeRole"
        }**
    ]
}
```

2. for role rad-role, add a trusted entity entry similar to below (2nd entry. first entry refers to the trust with oidc provider). 
  ```
  {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::817312594854:oidc-provider/oidc.eks.us-west-2.amazonaws.com/id/67DDAC18D8C44CEDCF1C9719A8E9B866"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.us-west-2.amazonaws.com/id/67DDAC18D8C44CEDCF1C9719A8E9B866:sub": "system:serviceaccount:irsa-mt:irsa-mt",
                    "oidc.eks.us-west-2.amazonaws.com/id/67DDAC18D8C44CEDCF1C9719A8E9B866:aud": "sts.amazonaws.com"
                }
            }
        },
        **{
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::817312594854:role/qa-role"
            },
            "Action": "sts:AssumeRole"
        }**
    ]
}
  ```

At this point, we can use aws AssumeRole to manage resources using account specific information. rp (for recipes) and ucp will have code changes that make sure of aws assumerole to be able to use the right credential specific to the radius environment for managing the aws resources.  

```
  client := sts.NewFromConfig(cfg)
  assumeRoleArn := retrieveRoleARNFromProviderInEnv()
  // Assume the new role
  assumeRoleProvider := stscreds.NewAssumeRoleProvider(client, assumeRoleArn)
  // Create a new config with the new credentials
  cfg.Credentials = assumeRoleProvider
  cfg.Region = region
  // Create a new S3 service client
  s3client := s3.NewFromConfig(cfg)
  // Call the ListBuckets function
  output, err := s3client.ListBuckets(context.TODO(), &s3.ListBucketsInput{})
  :

```

### API design

```
type AwsIRSACredentialsProperties struct {
  // REQUIRED; base Role with which radius service-accounts are annotated.
	RoleARN *string

	// REQUIRED; The AWS credential kind
	Kind *AWSCredentialKind

	// REQUIRED; The storage properties
	Storage CredentialStoragePropertiesClassification

	// READ-ONLY; The status of the asynchronous operation.
	ProvisioningState *ProvisioningState
}
```


```
type IRSAProvider struct {
	// Region is the AWS region to use.
	Region string

	// AccountID is the AWS account id.
	AccountID string

  // RoleARN of role.
  RoleARN string
}
```

Based on this new model we have to update API calls. 

## User Experience/ CLI Design

`rad init` and `rad env update` commands should be updated to support AWS IRSA.
Also, `rad install kubernetes --set` should be able to set a AWS role-arn. 

### non interactive support
rad install kubernetes --set global.aws.IRSAroleARN=<roleARN>

rad env update --aws-account-id <aws-account-id> --aws-region <aws-region> --aws-role-arn <aws-role-arn>

### interactive rad init support
```
% rad init --full  
Enter an environment name 
>default 

% rad init --full  
Select your cloud provider                     
1. Azure                                  
> 2. AWS                                     
3.[back] 

% rad init --full  

Select the identity for the AWS cloud provider 
1. access key id and secret                               
> 2. IRSA  

% rad init --full  

Enter the AWS RoleARN for configuring IRSA: <role-arn-radius>

```

This role-arn-radius should be used to annotate the radius ucp and rp service-accounts. When a cluster has the aws pod identity webhook installed, it will inject the necessary environment variables into pods associated with the service account and also mount the token as a secret. The only purpose for this role is to setup Radius services with IRSA. The policies associated with this role has no effect on deployment permissions. In otherwords, this account need not have any policy associated with it. 

Once radius is installed, we use rad env update to update the aws provider config
```
% rad env update --aws-account-id <aws-account-id> --aws-region <aws-region> --aws-role-arn <aws-role-arn>
```

The aws-role-arn corresponds to arn of the role which has the right policies for that environment. 



**Sample Output:**

n/a

**Sample Recipe Contract:**

n/a

### CLI Design

`rad init --full` will need to be updated to ask the user to choose between AWS access keys and IRSA.

We might want to rename `rad credential register aws` command  to `rad credential register aws access-keys` and introduce `rad credential register aws irsa`

We should update the Radius Helm chart to allow the user to enable IRSA during installation with the `global.awsIRSA.roleARN` value. This must be passed to all charts through values and when the value is set service accounts should be annotated with roleARN. 

### Implementation Details

#### UCP

UCP is responsible for communicating with AWS to deploy AWS resources. Therefore ucp's service-account will be annotated with AWS RoleARN.
Further UCP should be able to assume the role specified in the environment and use this to communicate with AWS.

#### Bicep

NA

#### Deployment Engine

NA

#### Core RP  / Recipes RP

Terraform provider requires AWS credentials in order to deploy recipes.
Therefore rp's service-account will be annotated with AWS RoleARN.

RP should be able to assume the role specified in the environment and use this to communicate with AWS.



### Error Handling
1. what if the webhook is not installed
2. what if the IRSA is not setup
3. what if the user has aws credentials (secrets) configured and wants to switch to IRSA?

## Test plan



## Security



## Compatibility


## Monitoring and Logging

We will have the same monitoring and logging as today. We will not be adding any new instrumentation.

## Development plan

* Create POC for Radius + AWS (1 engineer, 0.5 sprint)  [complete] 
* Create and review technical design (1 engineer, 0.5 sprint) [in progress]
* Implement model changes (1 engineer, 0.5 sprint) 
* Implement changes in UCP, and Recipes RP (1 engineer, 1 sprint)
* Implement CLI and Helm chart changes (1 engineer, 1 sprint) 
  * with this all radius pods should ahve neccessary configurations injected.
* End-to-end testing and documentation (1 engineer, 1 sprint) 

## Open Questions



## Alternatives considered 

n/a

## Design Review Notes

<!--
Update this section with the decisions made during the design review meeting. This should be updated before the design is merged.
-->

