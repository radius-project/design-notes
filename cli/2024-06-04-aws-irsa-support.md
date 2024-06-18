# Radius AWS Workload Identity Support

* **Status**: In Review
* **Author**: Nithya Subramanian (@nithyatsu)

## Overview

A software workload such as a container-based application, service or script needs an identity to authenticate, access, and communicate with services that are distributed across different platforms and/or cloud providers. Radius uses AWS IAM (Identity and Access Management) credentials to deploy and access AWS resources today:
 
```
rad credential register aws --access-key-id <access-key-id> --secret-access-key <secret-access-key>
```
 
These IAM credentials, including the access key should be rotated regularly to reduce the chance of unauthorized access. A more secure option than using these credentials is to use IRSA.
 
IRSA (IAM Roles for Service Accounts) is used for authenticating applications running within Kubernetes pods. When we use IRSA, we associate an IAM role directly with a Kubernetes service-account, allowing the pods associated with the service-account to assume that role. IRSA does not require us to configure access key and  secret. It relies identity tokens (OIDC tokens) to authenticate with AWS services. These tokens have a default expiration (configurable). However, they automatically refresh when needed, transparent to the user.

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

As a user, I should be able to configure Radius with IRSA for managing AWS resources.

## Design

### High Level Design for IRSA

During installation, Radius should allow user to enable IRSA for AWS while conifguring AWS provider. Then the user should be able to enter the roleARN associated with desired policies which will be stored as the AWS credential.

At a high level, the full flow to setup IRSA looks as below:

1. configure the cluster with oidc provider. have the role-arn configured with desired policies ready.
2. take steps to allow  `rad-role-arn` and `eks cluster : namespace: service` to trust each other using aws portal.
3. use  `rad init` to configure radius with role-arn credential. (or use rad credential later on)
4. use `rad env update` to configure provider associated with the specific environment (`rad env update qa --aws-account-id <aws-account-id> --aws-region <aws-region>`). 
5. during rad deploy, radius assumes the role that is configured in the credential. 

Since oidc is configured for a cluster and not specific to a role-arn, the solution can be extended to work in multi-tenancy once we have multi credentials support. 

More details about each step is covered in [Detailed Design](#detailed-design)

IRSA works the same way on EKS as well as non-EKS clusters.  

### Alternatives considered

#### EKS Pod Identity 

EKS Pod Identity was introduced in 2022 as a simplified approach for applications running on EKS to retrieve credentials. It uses the new EKS pod identity instead of OIDC provider for identity. Because of this, users have convenient APIs that allow managing pod identity. 
Ref. [AWS EKS Pod Identity](https://aws.amazon.com/blogs/containers/amazon-eks-pod-identity-a-new-way-for-applications-on-eks-to-obtain-iam-credentials/). 

Since the identity provider is tied to AWS, this solution cannot be used with Radius deployed on stand alone, aks or any other kubernetes cluster. 

Its good to note that since this approach uses its own service principal, the setup involves different identity providers and configuration on aws. Radius should be however able to support this feature with minimal code changes, if any.  


### Architecture Diagram


### Detailed Design

#### Proposed Option

Radius should support AWS IRSA.

#### Prerequisite

1. [Setup OIDC provider] (https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html) should be complete for the cluster. 
2. `rad-role` with desired policies should be created. Its trust relation should look similar to below. This establishes a trust relation between the annotated service-accounts in the specified namespace in specified cluster with the specified OIDC provider to be established. 
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
                    "oidc.eks.us-west-2.amazonaws.com/id/67DDAC18D8C44CEDCF1C9719A8E9B866:sub": "system:serviceaccount:radius-system:ucp",
                    "oidc.eks.us-west-2.amazonaws.com/id/67DDAC18D8C44CEDCF1C9719A8E9B866:aud": "sts.amazonaws.com"
                }
            }
        },
    ]
}
  ```

  Note: 67DDAC18D8C44CEDCF1C9719A8E9B866 is the specific eks cluster instance.
  The aud is the intended recipient of the token.
  The sub is the entity that the token represents.


#### Radius support

**installation**

User can enable IRSA for AWS by by choosing "IRSA" option by choosing IRSA during rad init and providing a RoleARN (`arn:aws:iam::123456789012:role/rad-role`). This stores the roleARN as credential. It also mounts the service-account token to ucp and rp pods as secret.


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
        Mounts:
          /etc/config from config-volume (rw)
          /var/run/secrets/eks.amazonaws.com/serviceaccount from aws-iam-token (ro)
          /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-d22ld (ro)
          /var/tls/cert from cert (ro)
```

Note: Since AWS IRSA works by mounting the service-account token to pods, the pod spec will be patched and pods recreated if we choose to use rad register to register the roleARN after radius installation.

**post installation**

Once the installation is complete, the user can use rad env update to store relevant information for deploying resources to accounts and region associated with specific radius environment.

```
rad env update qa --aws-account-id <aws-account-id> --aws-region <aws-region> 
```

At this point, we can use aws AssumeRole to manage resources using environment specific account information. rp (for recipes) and ucp will have code changes that make sure of aws assumerole to be able to use the right credential specific to the radius environment for managing the aws resources.  

```
  client := sts.NewFromConfig(cfg)
  assumeRoleArn := retrieveRoleARNCredential() //has logic to get the roleARN stored as credential
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
@doc("AWS credential kind")
enum AWSCredentialKind {
  @doc("The AWS IRSA credential")
  IRSA,
}
```

```
model AwsAccessKeyCredentialProperties extends AwsCredentialProperties {
  @doc("IRSA kind")
  kind: AWSCredentialKind.IRSA;
 
  @doc("The roleARN")
  roleARN: string;

  @doc("The storage properties")
  storage: CredentialStorageProperties;
}
```

Based on this new model we have to update API calls. 

## User Experience/ CLI Design

`rad init` and `rad credential` commands should be updated to support AWS IRSA.
Also, `rad install kubernetes --set` should be able to set a AWS role-arn. 

### non interactive support
rad install kubernetes --set global.aws.IRSA.enabled=true #default false

if enabled, it should print a success/ failure message after updating the pod spec to mount the token. It should also print a information message to the complete of configuration by using rad credential register and rad env update.

rad credential register aws --type <IRSA> --role <roleARN>
rad credential register aws --type <access-key> --access-key-id <access-key-id> --secret-access-key <secret-access-key>

rad env update --aws-account-id <aws-account-id> --aws-region <aws-region> 

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
1. access key                              
> 2. IRSA  

% rad init --full  

Enter the AWS RoleARN for configuring IRSA: <role-arn-radius>

```


Once radius is installed, we use rad env update to update the aws provider config
```
% rad env update --aws-account-id <aws-account-id> --aws-region <aws-region> 
```

**Sample Output:**

n/a

**Sample Recipe Contract:**

n/a

### CLI Design

`rad init --full` will need to be updated to ask the user to choose between AWS access keys and IRSA. rad credential register aws should now support two credential types, and associated parameters.

We should update the Radius Helm chart to allow the user to enable IRSA during installation with the `rad install kubernets --set global.aws.IRSA.enabled=true` value. This must be passed to ucp and rp charts through values and when the value is set the pod spec should mount the service-account token.

### Multi-tenancy

Currently, we support only one credential. However, for future, we might want to support multiple credentials potentially across AWS accounts ( say, one for QA team and other for Dev). The code changes for IRSA should be compatible with these changes in future. Since IRSA setup establishes trust between the cluster - oidc provider and AWS, once we support multiple credentials, we should be able to assume different roleARN based on which environment we are deploying from. We just have to establish right trust entities in all roles.  

### Implementation Details

#### UCP

UCP is responsible for communicating with AWS to deploy AWS resources. 
UCP should be able to assume the role specified in the credential and use this to communicate with AWS since it has the service-token mounted.

#### Bicep

NA

#### Deployment Engine

NA

#### Core RP  / Recipes RP

Terraform provider requires AWS credentials in order to deploy recipes.

RP should be able to assume the role specified in the credential and use this to communicate with AWS since its pod will have the service-token mounted.

### Error Handling

1. handle mismatch of credential type and arguments supplied.
2. what if the cluster does not have a oidc provider configured?
3. what if the user has aws credentials (secrets) configured and wants to switch to IRSA? we can print a informational message requesting user to delete the old credentials since we currently support only one credential. 

## Test plan

We should add a E2E to test the deployment of AWS resources using IRSA. 

## Security



## Compatibility


## Monitoring and Logging

We will have the same monitoring and logging as today. We will not be adding any new instrumentation.

## Development plan

* Create POC for Radius + AWS (1 engineer,1 sprint)  [complete] 
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



