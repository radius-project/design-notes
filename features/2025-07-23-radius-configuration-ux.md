# Radius Configuration UX

Today, Radius has very few system configurations. The few configurations it does have are set by the `rad init` setup experience. As Radius matures, there will many more additional configurations. It is important that these additional configurations are consistent and expected by users in order to provide a high quality user experience. This document examines other comparable platforms and proposes a set of recommendations for configuring Radius.

## Terms

**Imperative commands** – Imperative commands are CLI commands that give instructions or make requests to a system to perform an action. They are characterized by using a verb and a noun. The verb is typically create, read or show, update, delete, or list and the noun is typically a configuration for the system. For example, a user can imperatively create a Kubernetes pod via `kubectl run <pod-name> --image=<image-name>`.

**Declarative commands** – Declarative commands are CLI commands that tell a system the desired state. Declarative commands do not specify how, therefore, they only include a noun and do not include a verb. For example, a user can declaratively create a Kubernetes pod via `kubectl apply <pod-spec-yaml>`.

**Configuration** – An object which contains properties which describe the preferred behavior of the system. Sometimes referred to as a configuration resource, a configuration item, a preference, or an option. Within Radius, a configuration is modeled as a resource.

## Configuring Radius

Today, Radius has the following configurations:

* **Credentials** – Credentials are created imperatively via `rad credentials register` and `rad credentials unregister` commands. Unlike other Radius configurations, the register verb is used. This difference is discussed in the recommendations section below.
* **Environments** – Environments are created and deleted imperatively via `rad environment` commands or declaratively via `rad deploy env.bicep`. The declarative approach is required because the Environment resource is complex. It's simply not practical to include all the options in a single `rad environment create` command. The Environment has properties for:
  * Recipes for each Resource Type
  * Terraform configuration
  * Deployment locations including Kubernetes namespace, AWS account and region, and Azure subscription and resource group
  * Workload identity configuration
* **Resource Groups** – Resource Groups are created or deleted imperatively via `rad group` commands. Since Resource Groups do not have properties, creating and deleted Resource Groups is very easy via `rad group` commands.
* **Recipes** – Recipes are not a configuration in Radius. Rather, they are a property of the Environment. Environments are updated with recipes via the `rad recipe register` command rather than the `rad environment update` command. This is discussed further in the recommendations section below. Given recipes are a property of Environments, they are lower-cased in the remainder of this document, while proper configurations, such as Environments, are capitalized.
* **Resource Types** – Created and deleted imperatively via `rad resource-type` commands. The `rad resource-type create` command accepts a YAML file today and will support TypeSpec files in the future.
* **Workspaces** – Workspaces are created imperatively via `rad workspace` commands.

In the future, we expect several additional configurations including:

* Terraform 
* Recipe Packs
* Role Definitions
* Role Assignments

> [!IMPORTANT]
>
> This document makes broad assumptions about future configurations which are currently under design. Recipe Packs and Terraform may or may not be modeled as configurations. If they are, they will follow the proposals in this document.

## Comparisons

We examined how configurations are performed in AWS, Azure, and Kubernetes. Creating a role definition was used as a canonical example. Full examples are in the appendix, however, the conclusions can be summarized as:

* AWS, Azure, and Kubernetes offer both imperative and declarative approaches for creating configurations. Azure and Kubernetes are the most straightforward, while AWS is more complex and inconsistent by mixing up imperative and declarative commands. Azure and Kubernetes each have imperative commands for creating, reading, updating, and listing configurations. They both also have a single declarative command (`az deployment` and `kubectl apply`).
* In both imperative and declarative approaches, there is a one to one relationship between the configuration which is provided and what is created. While the role definition example is obviously straightforward, no other examples of configurations could be identified which took a user input and transformed it into something different. All declarative commands followed this principle and applied/deployed exactly what the user provided.
* For configurations that are global, Azure and Kubernetes handles them differently. Kubernetes puts all configurations in a namespace, even if they are global (`kubectl apply clusterRole.yaml -n my-ns` for example creates a global role definition but stores that configuration in the my-ns namespace). Azure takes the opposite approach (`az deployment sub create` and deploys configurations at the subscription level or group level).
* Kubernetes is the most consistent. Imperative commands take a subset of the properties as command line arguments but do not have input option. If a file input is required, the declarative command is used. For example, the command to create a cluster role only has verb and resource; it does not take a file input (`kubectl create clusterrole NAME --verb=verb --resource=resource.group`).

## Proposal for Radius

The following is proposed for Radius configurations:

### Imperative configuration

Radius will have both imperative and declarative commands for creating and updating configurations. The imperative commands follow these requirements:

* Each configuration will have imperative commands in the form: `rad <configuration> [create | update | delete | list | show]`. For example, when we add a configuration for Terraform, there will be a `rad terraform create` command. 
* Imperative commands will have up to five arguments for the most common properties for a configuration. If there are more than five arguments, the command will accept a YAML, JSON, or TypeSpec file as input. This aligns with the Azure behavior and departs from the Kubernetes behavior.
* Singleton configurations are unnamed while configurations with multiple instances are named. For example, `rad credential` are unnamed since there is only one.
* All configurations will be able to be set imperatively except recipes which are considered properties of Environments not actual configurations.

### Declarative configuration

Radius will support declarative commands for all configurations except Resource Types which will use a hybrid imperative/declarative since it bootstraps the Resource Type system. This makes repeated configurations of Radius easy and enables GitOps-based configuration of Radius.

* Configurations will be created declaratively using the `rad deploy` command which takes a Bicep file as input. Configurations will be modeled as Radius Resource Types using the `Radius.System` namespace.
* Configurations will be deleted declaratively via a new `rad delete` command which takes a Bicep file as input. 
* In order to support declarative creation and deletions of configurations, configurations must be stored in Radius as specified by the user. When a configuration is deployed using `rad deploy`, Radius must not perform post-deployment processing to transform the deployed configuration. For example, if a Recipe Pack configuration is deployed using `rad deploy`, it must be deletable via `rad delete`. Radius must not transform the Recipe Pack into multiple recipes after deployment.
* Unlike Azure, configurations which are global are still deployed to a resource group. Radius does not have a default group today when installed via `rad install`, but it should in the future.

### Standalone configuration versus property of another configuration

Configurations are modeled using typical data modeling techniques. A configuration is modeled as a standalone configuration when:

* The configuration is distinctive. For example, a Terraform configuration is certainly distinct from an Azure credential in that they are not related to each other. It would not semantically make sense for Terraform to be a property of a Credential.
* The cardinality of the two configurations are different. Continuing with the Terraform example, there is one configuration for the Terraform provider mirror, while there are hundreds of Environments. It does not make sense to have hundreds of Terraform configurations therefore the Terraform configuration should be modeled as a standalone configuration.

### Internal versus external configurations

Radius will store all configurations within the Radius control plane. No configuration will be stored externally. For example, Radius Role Definitions will be stored as Role Definitions created with the `rad role-definition create` rather than as a YAML stored in a Git repository. The benefits of this is that Radius does not take a dependency on an external system. Should Radius lose access to the Git repository (possible after a credential expires), Radius will not function correctly. 

The downside of this approach is that it complicated GitOps approaches where configurations are stored in Git. However, that is the role of CD tools such as ArgoCD and Flux and also is why Radius must offer the ability to maintain configurations declaratively.

Recipes may be considered an exception to this rule. However, rather being an exception, Recipes are considered external code to be executed, not Radius configurations.

#### Register versus create

The register verb is used by Radius today for recipes and Credentials. The register verb is used to denote that Radius is being made aware of an external resource rather than actually creating that resource within the Radius control plane. The use of the register verb should be used sparingly given all configurations are stored within Radius. 

Registering recipes is a valid use of the register verb. However, registering credentials is the incorrect use. The `rad credential register` command indeed creates a credential within the Radius control plane and does not reference an external resource. Renaming the command to `rad credential create` is not being proposed here, but it is important to note this as an exception to the intended use of the register verb.

## Existing Radius Configurations

| Configuration | Imperative Command                                           | Declarative Command                                          |
| ------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Credential    | ✅ `rad credential [register\|show\|unregister]`              | ❌ Not possible today                                         |
| Environment   | ✅ `rad environment [create\|delete\|list\|show\|update]`     | ✅ `rad deploy` with `Applications.Core/environments` resource ❌ No `rad delete` |
| Group         | ✅ `rad group [create\|delete\|list\|show]` (There is no update command since there are not properties.) | ❌ Not possible today                                         |
| recipe        | ❌ `rad recipe [list\|register\|show\|unregister]` (This is a deviation from the Radius design principle since recipe is a property of the Environment.) | ✅ See Environment                                            |
| Resource Type | ✅ `rad resource-type [create\|delete\|list\|show]`           | ✅ N/A                                                        |
| workspace     | ✅ `rad workspace [create\|delete\|list\|show]` (This is an exception since workspace is not a Radius configuration, but rather the local CLI configuration.) | ✅ N/A                                                        |

## Upcoming Radius Configurations

There are several configurations which are being planned in the near future.|

* **Terraform** – Configuration of the Terraform CLI including the installation binary location, provider mirrors, module locations, and associated credentials
* **Recipe Packs** – A bundle of recipes added to the Environment configuration
* **Role Definitions** – A role with permissions to perform a defined set of actions on a defined set of resources
* **Role Assignments** – A mapping of a role definition to a user

The table below summarizes the imperative and declarative commands for these new configurations.


| Configuration | Imperative Command | Declarative Command |
| ------------- | ------------------ | ------------------- |
| Terraform | `rad terraform [create\|update\|list\|show\|delete]` | `rad deploy` and `rad delete` with `Radius.System/terraform` resource |
| Recipe Pack | `rad recipe-pack [create\|update\|list\|show\|delete]` | `rad deploy` and `rad delete` with  `Radius.System/recipePacks` resource |
| Role Definition | `rad role-definition [create\|update\|list\|show\|delete]` | `rad deploy` and `rad delete` with  `Radius.System/roleDefinition` resource |
| Role Assignment | `rad role-assignment [create\|update\|list\|show\|delete]` | `rad deploy` and `rad delete` with  `Radius.System/roleAssignment` resource |

## Appendix – Comparisons

### AWS

AWS offers both imperative and declarative methods for creating IAM roles, however, they are extremely similar.

#### Imperative methods:

```bash
$ cat s3-read-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-example-bucket",
        "arn:aws:s3:::my-example-bucket/*"
      ]
    }
  ]
}

$ aws iam put-role-policy --role-name MyEC2Role --policy-name MyEC2S3ReadPolicy --policy-document file://s3-read-policy.json
```

#### Declarative methods:

```bash
$ cat s3-iam-reader.yaml
# This CloudFormation template defines an IAM Role with an embedded inline policy.
Resources:
  MyIAMRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: MyS3ReadRole  # You can choose a different name for your role
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
        - Effect: Allow
          Principal:
            Service: ec2.amazonaws.com # Example: allows EC2 instances to assume this role
          Action: sts:AssumeRole
      Policies:
        - PolicyName: S3ReadOnlyAccessPolicy # Name for the inline policy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
            - Effect: Allow
              Action:
                - s3:GetObject
                - s3:ListBucket
$ aws cloudformation deploy --stack-name MyDeclarativeRoleStack --template-file s3-iam-reader.yaml
```

### Azure

#### **Imperative methods:**

* Via the Azure portal where the user gets presented with a GUI specific to creating a role definition
* Via PowerShell or the CLI:

```bash
$ az role definition create --role-definition ~/roles/vmoperator.json
$ az role definition list --name "Virtual Machine Operator"
$ az role definition update --role-definition ~/roles/vmoperator.json
$ az role definition delete --name "Virtual Machine Operator"
```

#### **Declarative methods:**

* Via Bicep or ARM:

```bash
$ cat roleDefinition.bicep
resource roleDef 'Microsoft.Authorization/roleDefinitions@2022-04-01' = {
  name: roleDefName
  properties: {
    roleName: roleName
    description: roleDescription
    type: 'customRole'
    permissions: [
      {
        actions: actions
        notActions: notActions
      }
    ]
    assignableScopes: [
      subscription().id
    ]
  }
}
# `sub` means deploy a resource at the subscription level
$ az deployment sub create --location eastus --name customRole --template-file roleDefinition.bicep 
$ az deployment sub delete --name <deployment-name>
```

A common challenge for declarative configurations is that some configurations apply at one level in the configuration hierarchy while others apply at another level. Radius, for example, has global configurations (Resource Types and credentials) and environment-level configurations (recipes). Azure tackles this challenge by using different scopes on the deployment command—the `az deployment sub` command to create configurations at the subscription level while `az deployment group` creates configurations at the resource group level.

### Kubernetes

Kubernetes also tackles the scoping differently and uses different Resource Types for different scopes. For example, a role  is scoped to the namespace while a cluster role is scoped to the cluster. However, a cluster role can be deployed/created in any namespace.

#### **Imperative method:**

```bash
$ kubectl create clusterrole pod-reader --verb=get --verb=list --verb=watch --resource=pods
$ kubectl get clusterroles
$ kubectl describe clusterrole pod-reader
$ kubectl edit clusterrole
$ kubectl delete clusterrole
```

Kubernetes has a nice feature which allows a configuration to be edited in place. `kubectl edit` opens the resource manifest in the default editor, `vi` for example.

#### **Declarative method:**

```bash
$ cat pod-reader.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
$ kubectl apply pod-reader.yaml
$ kubectl delete -f pod-reader.yaml
```

