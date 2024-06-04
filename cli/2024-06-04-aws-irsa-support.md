# Radius Azure Workload Identity Support

* **Status**: In Review
* **Author**: Nithya Subramanian (@nithyatsu)

## Overview

A software workload such as a container-based application, service or script needs an identity to authenticate, access, and communicate with services that are distributed across different platforms and/or cloud providers. Radius uses AWS IAM (Identity and Access Management) credentials to deploy and access AWS resources.
 
 ```
rad credential register aws --access-key-id <access-key-id> --secret-access-key <secret-access-key>
 ```
 
These IAM credentials, including the access key should be rotated regularly to reduce the chance of unauthorized access. A more secure option than using these credentials is to use IRSA.
 
IRSA (IAM Roles for Service Accounts) is used for authenticating applications running within Kubernetes pods. When we use IRSA, we associate an IAM role directly with a Kubernetes service account, allowing the pods to assume that role. While IRSA does not use access key, it relies identity tokens (OIDC tokens) to authenticate with AWS services. These tokens have a default expiration (usually 15 minutes). However, they automatically refresh when needed, transparent to the user.

The goal of the scenario is to enable infrastructure operators to configure IRSA support for the AWS provider in Radius to deploy and manage AWS resources.

## Terms and definitions



## Objectives

> Issue Reference: https://github.com/radius-project/radius/issues/7618 

### Goals

* Radius users can configure AWS provider to use IRSA for authentication.
* IRSA can be configured via interactive experience.
* IRSA can be configured manually.
* Radius users can deploy and manage AWS resources without using AWS access key id and seceret

### Non-goals

* Azure Managed Identity support
* Azure Workload Identity support

### User scenarios

As a user, I should be able to use IRSA to provide Radius with access to AWS resources.

## User Experience

rad init should be updated with prompts that allow user to pick a configure  AWS IRSA. We would require account ID and region (for AWS provider) and roleARN for installing radius pods with IRSA support.

rad install kubernetes --set global.aws.IRSAroleARN=<roleARN>


**Sample Output:**

See above

**Sample Recipe Contract:**

n/a

## Design

### High Level Design

During installation, user can enable IRSA for AWS by by choosing "IRSA" option while configuring AWS provider and providing the Role ARN.
Then, service-accounts will be annotated with role arn, similar to `amazonaws.com/role-arn: arn:aws:iam::123456789012:role/radius-role`. A trust relation between the annotated service-accounts in the specified namespace with the specified OIDC provider is established.

[Amazon EKS Pod Identity Webhook](https://github.com/aws/amazon-eks-pod-identity-webhook#amazon-eks-pod-identity-webhook) (to be setup in cluster by user as part of IRSA setup ref: [AWS IRSA documentation](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) will inject neccessary configurations into the pods associated with these service-accounts for IRSA.  

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
          AWS_ROLE_ARN:                 arn:aws:iam::817312594854:role/my-role
          AWS_WEB_IDENTITY_TOKEN_FILE:  /var/run/secrets/eks.amazonaws.com/serviceaccount/token
        Mounts:
          /etc/config from config-volume (rw)
          /var/run/secrets/eks.amazonaws.com/serviceaccount from aws-iam-token (ro)
          /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-d22ld (ro)
          /var/tls/cert from cert (ro)
```

Then, 

We can use these environment variables in our code similar to below snippet to retrieve credentials. 

```
    region := "us-west-2"
    roleArn := os.Getenv("AWS_ROLE_ARN")
    tokenFilePath := os.Getenv("AWS_WEB_IDENTITY_TOKEN_FILE")
    
    cfg, err := config.LoadDefaultConfig(context.TODO(), config.WithRegion(region))
    if err != nil {
        panic("failed to load config, " + err.Error())
    }

    client := sts.NewFromConfig(cfg)

    credsCache := aws.NewCredentialsCache(stscreds.NewWebIdentityRoleProvider(
        client,
        roleArn,
        stscreds.IdentityTokenFile(tokenFilePath),
        func(o *stscreds.WebIdentityRoleOptions) {
            o.RoleSessionName = "rad-session"
        }))

    creds, err := credsCache.Retrieve(context.TODO())

    // fmt.Printf("AccessKeyID: %s\n", creds.AccessKeyID)
	// fmt.Printf("SecretAccessKey: %s\n", creds.SecretAccessKey)
	// fmt.Printf("SessionToken: %s\n", creds.SessionToken)
	// fmt.Printf("Expires: %v\n", creds.Expires)
	// fmt.Printf("Source: %s\n", creds.Source
```


### Architecture Diagram



### Detailed Design

There are a few ways a workload can access AWS resources in addition to directly configuring workloads with access key/secrets.  

IRSA  

IRSA allows us to associate an IAM role directly with a Kubernetes service account and a namespace. When a pod assumes an IAM role, it can access AWS services using temporary credentials provided by the role. These credentials are automatically rotated and managed by AWS. IRSA works the same way on EKS as well as non-EKS clusters.  

EKS Pod Identity 

EKS Pod Identity was introduced in 2022 as a simplified approach for applications running on EKS to retrieve credentials. It does not need setting up an OIDC identity provider and achieves this simplicity by using a new EKS service principal and APIs. 

#### Advantages (of each option considered)


IRSA is well established and works well with EKS as well as other cluster including self managed clusters. This makes it ideal for Radius despite the configuration overhead.

EKS Pod Identity makes the configuration simpler, but works only on EKS workloads. 


#### Disadvantages (of each option considered)

With IRSA, the user has to setup OIDC provider and configure appropriate accesses. He may have to coordinate with other administrators to acheive this.
However, we can not alleviate this by using EKS pod identity since we want a vendor neutral approach.

#### Proposed Option

we will document design details for Radius to support AWS IRSA.

### API design

We currently use account ID, access key for AWS credential.

```
type AwsAccessKeyCredentialProperties struct {
	// REQUIRED; Access key ID for AWS identity
	AccessKeyID *string

	// REQUIRED; The AWS credential kind
	Kind *AWSCredentialKind

	// REQUIRED; Secret Access Key for AWS identity
	SecretAccessKey *string

	// REQUIRED; The storage properties
	Storage CredentialStoragePropertiesClassification

	// READ-ONLY; The status of the asynchronous operation.
	ProvisioningState *ProvisioningState
}
```
But since the environment variables in all pods are populated with neccessary information for IRSA, we do not need a specific credential for IRSA. 

We have to update AWS provider  to allow AccessKeyID and SecretAccessKey to be optional.

```
( I am not sure why we have the secrets here we need only account no and region to build scope)
type Provider struct {
	// AccessKeyID is the access key id for the AWS account.
	AccessKeyID string

	// SecretAccessKey is the secret access key for the AWS account.
	SecretAccessKey string

	// Region is the AWS region to use.
	Region string

	// AccountID is the AWS account id.
	AccountID string
}
```

Based on this new model we would update API calls. ( to double check)



### CLI Design

`rad init --full` will need to be updated to ask the user to choose between AWS access keys and IRSA.

We might want to rename `rad credential register aws` command  to `rad credential register aws access-keys` and document this is not required for irsa.

We will need to update the Radius Helm chart to allow the user to enable IRSA during installation with the `global.awsIRSA.roleARN` value. This must be passed to all charts through values and when the value is set service accounts should be annotated with roleARN. 
# ----------------------------------------------------------------------------

### Implementation Details

#### UCP

UCP should be updated to craete a credential based on RoleARN and service access token, when teh environment variables are set. It should then authenticate with AWS using this.

#### Bicep

n/a

#### Deployment Engine

Deployment Engine should be updated to craete a credential based on RoleARN and service access token, when teh environment variables are set. It should then authenticate with AWS using this.

#### Core RP

?

#### Portable Resources / Recipes RP

n/a

### Error Handling



## Test plan



## Security



## Compatibility


## Monitoring and Logging

We will have the same monitoring and logging as today. We will not be adding any new instrumentation.

## Development plan

* Create POC for Radius + AWS (1 engineer, 0.5 sprint)  
* Create and review technical design (1 engineer, 0.5 sprint)
* Implement CLI and Helm chart changes (1 engineer, 1 sprint)
  * with this all radius pods should ahve neccessary configurations injected.
* Implement changes in UCP, Bicep, and Applications RP (1 engineer, 1 sprint)
  * use aws-sdk-2 to create credentials from IRSA settings.
* End-to-end testing and documentation (1 engineer, 0.5 sprint)

## Open Questions



## Alternatives considered 

n/a

## Design Review Notes

<!--
Update this section with the decisions made during the design review meeting. This should be updated before the design is merged.
-->

TODO