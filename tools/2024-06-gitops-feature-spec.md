# Radius Integration with GitOps

* **Status**: Pending
* **Author**: Will Tsai (@willtsai)

## Summary
<!-- A paragraph or two to summarize the Epic. Just define it in summary form so we all know what it is. -->
Continuous deployment of cloud native applications and infrastructure is challenging for a number of reasons.  GitOps is a popular set of practices, implemented as popular tools, like Flux and ArgoCD, that mitigates these challenges for enterprise application teams that use git for source control and Kubernetes for orchestration of software containers.  GitOps provides a developer-centric experience by abstracting and automating many of the tasks required for deploying and operating Kubernetes and its underly dependent infrastructure. The core concept of GitOps is to rely on a git repository that serves as a single source of truth: i.e. it contains current declarative descriptions of the required infrastructure for a given production environment. It also contains a description of the workflow required to prevent drift between the repo and the production environment. It's like having cruise control for managing applications in production.

Enterprises that use GitOps and also want to use Radius do not currently have a clear path for how to use both technologies in an integrated and complementary way to enable a “better together” experience.

### Top level goals
<!-- At the most basic level, what are we trying to accomplish? -->
- Enable enterprises to get the best of both GitOps and Radius for a better-together experience.
- Provide a consistent Radius + GitOps integration model for both existing and future GitOps platforms.

### Non-goals (out of scope)
<!-- What are we explicitly not trying to accomplish? -->
- We are not trying to replace GitOps tools or platforms.  We are trying to integrate Radius with existing GitOps tools and platforms.
- We are not trying to replace GitOps practices.  We are trying to enable Radius to work well with GitOps practices.

## Customer profile and challenges
<!-- Define the primary customer and the key problem / pain point we intend to address for that customer. If there are multiple customers or primary and secondary customers, call them out.   -->

### Customer persona(s)
<!-- Who is the target customer? Include size/org-structure/decision makers where applicable. -->
**Infrastructure operators / administrators** - In enterprise applications team that use GitOps, operators are responsible for configuring and managing the git repos for infrastructure and application configuration. They configure GitOps policies as well as the Kubernetes manifests, Helm charts, etc. that are applied to the application and the Kubernetes cluster.

**Application developers** - Responsible for designing, developing and maintaining application code. Developers use GitOps by checking files into their git repositories to configure GitOps settings to auto deploy and monitor their application and dependent infrastructure.

**Site Reliability Engineers** - They leverage GitOps by checking in configuration files into their git repositories that add information for the customization of the workload, i.e. replica pod number, strategic merge solutions. 

### Challenge(s) faced by the customer
<!-- What challenges do the customer face? Why are they experiencing pain and why do current offerings not meet their need? -->
Before the introduction of GitOps, enterprise application teams struggled with complex deployment processes; environment/application configuration drift; as well as difficulty following best practices for security and disaster recovery.  GitOps enables teams to define continuous deployment workflows and automation to simplify many of these tasks, which add significant value by increasing efficiency, security, creating better developer experiences, and allowing for faster deployments between code changes.

Radius offers additional complementary value for enterprise application teams, including a rich, declarative application description which enables the Radius application graph.  Radius also provides a clear separate of concerns via Recipes.  However, without integration of Radius and GitOps, plus clear guidance on how to use the two together, enterprises already invested in GitOps don't know how to use both GitOps and Radius for a better together experience.

<!-- What is the positive outcome for the customer if we deliver these features, i.e. what is the value proposition for the customer? Remember, this is customer-centric. -->

## Key scenarios
<!-- List ~3-7 high level scenarios to clarify the value of the Epic and point to how we will decompose this big area into component capabilities. We may ultimately have more than one level of scenario. -->

### Scenario 1: As an operator I can install and configure Radius with my GitOps tool of choice
<!-- One or two sentence summary -->
My team uses GitOps to manage Kubernetes clusters and applications that run in those clusters.  All relevant configuration information is stored in a git repository that serves as the source of truth for both the application code and infrastructure configuration.  To use Radius, I need to include Radius as a dependency for my Kubernetes cluster, just as I would any other dependency, via my GitOps tools.

### Scenario 2: As an operator or developer, I can deploy and manage Radius Environments, Recipes, applications, and resources via my GitOps toolset
<!-- One or two sentence summary -->
With Radius installed, as an Operator, I now define, deploy and manage a Radius environment and associate relevant Recipes to that environment. 

As a Developer, my Operations counterpart has provided a Radius Environment plus Recipes to enable my Radius application development.  I have familiarized myself with Radius and have experimented with deploying my application to this environment using the Recipe.  After my app is deployed, I can view its graph in the Radius dashboard.  Everything is working well for me so far and I really appreciate these cool Radius features! 

### Scenario 3: As a Site Reliability Engineer (SRE), I can patch and edit the configuration in my Kubernetes cluster through my GitOps toolset.
<!-- One or two sentence summary -->
As an SRE, my Operations counterpart has provided a Kubernetes cluster, a Radius Environment running on that cluster and the infrastructure resources through Radius Recipes required for the Radius Applications that will run in that Radius Environment. I am expected to make adjustments to the configuration of the Kubernetes cluster and the Radius Environment as needed. I can do this through my GitOps toolset.

### Stretch Goal: As an Operator and as a Developer, I can visualize the changes tracked by my GitOps toolset on my Radius Dashboard
We still need to flesh this out but the basic idea is the Radius dashboard can leverage the Notification Controller in Flux and ArgoCD to display to users inbound and outbound information of the source changes and cluster changes affecting their application and resources.

## Key dependencies and risks
<!-- What dependencies must we take in order to enable this scenario? -->
<!-- What other risks are you aware of that need to be mitigated. If you have a mitigation in mind, summarize here. -->
<!-- **Dependency Name** – summary of dependency.  Issues/concerns/risks with this dependency -->
<!-- **Risk Name** – summary of risk.  Mitigation plan if known. If it is not yet known, no problem. -->
Dependencies - Flux, ArgoCD, future GitOps tools/platforms. 

The primary risk, per the question below, is whether an abstracted, generalizable extensibility model for GitOps is feasible.

## Key assumptions to test and questions to answer
<!-- If you are making assumptions that, if incorrect, would cause us to significantly alter our approach to this scenario, make them explicit here.  Also call out how / when you plan to validate key assumptions. -->
<!-- What big questions must we answer in order to clarify our plan for this scenario.  When and how do you plan to answer those questions (prototype feature x, customer research, competitive research, etc) -->
Our goal is to deliver a generalized model such that Flux, ArgoCD plus future GitOps technologies can all be integrated in a consistent manner.  How feasible is that?  We'll need to learn more about the GitOps tool landscape to determine feasibility.

We as the Radius team understand that while Flux and ArgoCD might currently be the most popular options utilized by customers and while the current scenarios might reference Flux or ArgoCD, the solution we will build will be abstract enough to support all other high priority GitOps toolsets. We currently understand the GitOps workflow if abstracted in implementation usually follows this format:
1. Install GitOps tool onto Kubernetes cluster. 
2. Add git repository information to GitOps tool
3. Deploy application to Kubernetes cluster 
4. Done 
 
> The implementation settled on is generic enough to support GitOps on a wide level not narrowed down to just Flux or ArgoCD.

## Current state
TBD

# Scenario 1: Install and configure Radius using GitOps

## Target customers
<!-- Of the customers / personas listed in the Epic doc, what subset are we delivering this scenario to serve? -->
**Infrastructure operators / administrators** - In enterprise applications teams that use GitOps, operators are responsible for configuring and managing the git repos for infrastructure and application configuration. They configure GitOps policies as well as the Kubernetes manifests, Helm charts, etc. that are applied to the application and the Kubernetes cluster.

## Details of customer problem
<!-- <Write this in first person. You basically want to summarize what “I” as a customer am trying to accomplish, why the current experience is a problem and the impact it has on me, my team, my work and or biz, etc…. i.e. “When I try to do x aspect of cloud native app development, I have the following challenges / issues….<details>. Those issues result in <negative impact those challenges / issues have on your work and or business.> -->
As an infrastructure operator in an enterprise applications team that has to maintain complex Kubernetes clusters and the platforms that run on them, I need to ensure that my team can deploy and manage applications and infrastructure in a consistent and reliable way. Thus, we make use of GitOps to manage Kubernetes clusters and applications that run in those clusters. All relevant configuration information is stored in a git repository that serves as the source of truth for both the application code and infrastructure configuration. To use Radius, I need to include Radius as a dependency for my Kubernetes cluster, just as I would any other dependency, via my GitOps tools. Without clear guidance on how to do this, I am unsure how to use both GitOps and Radius for a better together experience.

## Desired customer experience outcome
<!-- <Write this as an “I statement” that expresses the new capability from customer perspective … i.e. After this scenario is implemented “I can do, x, y, z, steps in cloud native app developer and seamlessly blah blah blah …. As a result <summarize positive impact on your work / business>  -->
After this scenario is implemented, I can install and configure Radius with my GitOps tool of choice. I can include Radius as a dependency for my Kubernetes cluster, just as I would any other dependency, via my GitOps tools. This ensures that I no longer have to manually manage the Radius instance I am providing for my developers and SREs to use, that there is an automated process to ensure that the deployed Radius instance matches the state specified I have specified in a repository. In other words, my experience with managing our Radius instance is consistent with how I normally use GitOps to define any other dependency in a Kubernetes environment.

### Detailed Customer Experience
 <!-- <List of steps the customer goes through from the start to the end of the scenario to provide more detailed view of exactly what the customer is able to do given the new capabilities>  -->
<!-- Step 1
Step 2
… -->
1. With my editor of choice (such as VSCode) open the GitOps (Flux or ArgoCD) Source File  
1. Add installation of Radius as a Kubernetes cluster dependency for my desired Radius version.
   1. Similar the params in the Flux schema defined here: [Helm Charts | Flux (fluxcd.io)](https://fluxcd.io/flux/components/source/helmcharts/)
1. Commit the change to the git repo to submit my change and track it for future reference.
1. GitOps tool will detect this repo change and update the relevant Kubernetes clusters 
1. I can validate the Radius installation through various GitOps or Kubernetes tools. For example, I can run `kubectl get namespace` and verify Radius appears as Kubernetes namespace. 
   1. I will be able to validate Radius on my Flux dashboard or my rad CLI/GitOps CLI as well once I've created a Radius Environment or have Radius Applications running.

> As an operator, I'm happy with this experience because it is completely consistent with how I normally use Flux or ArgoCD to define any other dependency in a Kubernetes environment.
 
The syntax for adding Radius as a Kubernetes cluster dependency must be:
- Consistent with customer expectations for defining such a dependency 
- Executable by Flux or ArgoCD (which based on current understanding should just work today). 

> There is no new Radius specific user experience for this scenario. Customers are only interacting with Flux, ArgoCD, or their GitOps tool of choice.

## Key investments
<!-- List the features required to enable this scenario. -->
Based on current understanding, there should be no new features that need to be implemented in Radius for this scenario. The key investment is in testing the GitOps tooling to ensure that the syntax for adding Radius as a Kubernetes cluster dependency is consistent with customer expectations for defining such a dependency and is executable by Flux or ArgoCD.

## Key dependencies and risks
<!-- What dependencies must we take in order to enable this scenario? -->
<!-- What other risks are you aware of that need to be mitigated. If you have a mitigation in mind, summarize here. -->
<!-- Dependency Name – summary of dependency.  Issues/concerns/risks with this dependency -->
<!-- Risk Name – summary of risk.  Mitigation plan if known. If it is not yet known, no problem. -->

**Dependency: Flux, ArgoCD, etc.** - These GitOps tools/platforms must be able to detect the Radius dependency specified in the GitOps source file and update the relevant Kubernetes clusters accordingly to install Radius.

**Risk: ability for GitOps tools to install and deploy Radius** - The primary risk is whether Flux, ArgoCD, etc. can install and deploy Radius as a Kubernetes cluster dependency. We will need to test this to ensure that it works as expected.

## Key assumptions to test and questions to answer
<!-- If you are making assumptions that, if incorrect, would cause us to significantly alter our approach to this scenario, make them explicit here.  Also call out how / when you plan to validate key assumptions. -->
<!-- What big questions must we answer in order to clarify our plan for this scenario.  When and how do you plan to answer those questions (prototype feature x, customer research, competitive research, etc) -->

**Assumption:** The syntax for adding Radius as a Kubernetes cluster dependency must be consistent with customer expectations for defining such a dependency and must be executable by Flux or ArgoCD. We will need to test this to ensure that it works as expected.

# Scenario 2: Deploy and manage Radius Environments, Recipes, Apps, Resources using GitOps

## Target customers
<!-- Of the customers / personas listed in the Epic doc, what subset are we delivering this scenario to serve? -->
**Infrastructure operators / administrators** - In enterprise applications team that use GitOps, operators are responsible for configuring and managing the git repos for infrastructure and application configuration. They configure GitOps policies as well as the Kubernetes manifests, Helm charts, etc. that are applied to the application and the Kubernetes cluster.

**Application developers** - Responsible for designing, developing and maintaining application code. Developers use GitOps by checking files into their git repositories to configure GitOps settings to auto deploy and monitor their application and dependent infrastructure.

## Existing customer problem
<!-- <Write this in first person. You basically want to summarize what “I” as a customer am trying to accomplish, why the current experience is a problem and the impact it has on me, my team, my work and or biz, etc…. i.e. “When I try to do x aspect of cloud native app development, I have the following challenges / issues….<details>. Those issues result in <negative impact those challenges / issues have on your work and or business.> -->
Since my organization uses GitOps to manage Kubernetes clusters and applications that run in those clusters, I need to remain consistent in infrastructure and application deployment even when using Radius. This means that I need to be able to apply Radius-centric concepts like Environments, Recipes, Applications, and Resources in a way that is consistent with my existing GitOps workflows. Without clear guidance on how to do this, I am unsure how to use both GitOps and Radius for a better together experience.

## Desired customer experience outcome
<!-- <Write this as an “I statement” that expresses the new capability from customer perspective … i.e. After this scenario is implemented “I can do, x, y, z, steps in cloud native app developer and seamlessly blah blah blah …. As a result <summarize positive impact on your work / business>  -->
With the implementation of this scenario, I can deploy and manage Radius Environments, their associated Recipes, Applications, and Resources via my existing GitOps (Flux and ArgoCD) toolsets. This means that I don't have to author pipelines or manually run Radius commands to deploy Radius-managed resources. Instead, I can leverage a git repo as the source of truth for my Radius-managed resources and use my existing GitOps tooling to deploy and manage those resources. This ensures that my experience with managing Radius resources is consistent with how I normally use GitOps to manage Kubernetes clusters and applications.

> There is no new Radius specific user experience for this scenario. Customers are only interacting with Flux or ArgoCD.

### Detailed Customer Experience
 <!-- <List of steps the customer goes through from the start to the end of the scenario to provide more detailed view of exactly what the customer is able to do given the new capabilities>  -->
<!-- Step 1
Step 2
… -->

**Step 1: Define and deploy Radius Environment and Recipes**

As an operator, I need to define and deploy a Radius Environment and associate relevant Recipes to that environment so that my development team can build and deploy their Radius applications.

1. Using my editor of choice, such as VSCode, I open an existing Radius environment file, `env.bicep`, from the online Radius samples repo. 
1. I use Radius documentation to determine specific edits to my `env.bicep` file such as Kubernetes cluster and the appropriate cloud provider registrations. 
   1. I will create a Radius Environment resource and add all the information listed above. 
   1. I will then edit my Radius Environment resource to contain the properties required to register my Radius Recipe such as template path, template kind, etc.
1. I then push my new `env.bicep` file to my git repository, as I would any other file. 
1. This triggers Flux or ArgoCD to detect the change, read the new `env.bicep` file, and deploy the Radius Environment as specified.
1. After Flux or ArgoCD initiates the Radius environment deployment, as I would normally do, I use Flux or ArgoCD to monitor deployment status and health status of the underlying Kubernetes cluster. 

> As an operator, while Radius presents some new concepts, those concepts fit cleanly within my existing GitOps workflows so Radius is relatively low overhead for me to adopt and to provide value to my development counterparts.   

**Step 2: Deploy and manage Radius Applications and Resources through GitOps toolset**

Now, as a developer, I need to make some changes to my application code. Specifically, I need to change the Radius Resource for my frontend UI. My frontend resource is already running in my Radius application but I need update the code for parsing user input. First, I'll update the required file where this function lives then I'll update my Radius Resource as follows: 

1. With my editor of choice, such as VSCode, I open the existing `frontend` application file `helper.ts` and update my function `parseUsers` with the new required logic. 
2. Then I push the changes to my git repository. 
3. This triggers Flux/ArgoCD to detect the change and update the `frontend` resource with the new function. I use Flux/ArgoCD to monitor the health status of the Kubernetes cluster and immediately realize the pod associated with my `frontend` resource is failing.
4. I review my code change again and realize I created a bug in the function which is causing the resource to fail. 
5. To resolve, I:
   1. Use the Flux/ArgoCD rollback commands to restart my `frontend` resource pods with the previous code version so as not to disrupt my customers. 
   1. Proceed with a new code change to my `helper.ts` that resolves the bug, then push this change to my repository. 
6. Flux/ArgoCD now deploys my bug fix and I can see in Flux/ArgoCD that the pod has restarted in a healthy state with my new code. 

> As a Developer, I'm very happy with this experience because I get the cool new features of Radius (like self-serve infrastructure deployment via Recipes and the App Graph/Dashboard) in addition to the features I already know and love in Flux, like health monitoring and rollback.

__Requirements resulting from these scenarios:__ 
- Flux/ArgoCD must be able to detect the file change described above and must be able to read the env.bicep file as required to deploy the Radius app correctly 
- Radius types and Bicep files must be able to be read by Flux/ArgoCD which currently does not have that ability.
- Flux/ArgoCD must be able to detect Radius resource types and execute Flux/ArgoCD commands against those types, including rollback *list of any other required Flux commands*.
- Based on user feedback this could be just testing that commands such as dry run mechanisms work

## Key investments
<!-- List the features required to enable this scenario. -->

### Feature 1: GitOps tooling can detect and understand Radius definition files
<!-- One or two sentence summary -->
GitOps tools (beginning with Flux and ArgoCD) must be able to detect the file change in the git repo described above and must be able to read the `env.bicep` file as required to deploy the Radius app correctly.

GitOps tools (beginning with Flux and ArgoCD) must be able to detect changes to Radius applications and other resource type definitions in the git repo and execute the appropriate deployment, etc. commands against those types, including rollback operations.

### Feature 2: GitOps tooling can execute Radius commands
<!-- One or two sentence summary -->
Once Radius definition file changes are detected and understood, GitOps tools (beginning with Flux and ArgoCD) must be able to execute deployments for the Radius resources defined in the git repo.

### Feature 3: GitOps tooling can rollback Radius resources
<!-- One or two sentence summary -->
GitOps tools (beginning with Flux and ArgoCD) must be able to execute rollback operations for Radius resources defined in the git repo.

## Key dependencies and risks
<!-- What dependencies must we take in order to enable this scenario? -->
<!-- What other risks are you aware of that need to be mitigated. If you have a mitigation in mind, summarize here. -->
<!-- Dependency Name – summary of dependency.  Issues/concerns/risks with this dependency -->
<!-- Risk Name – summary of risk.  Mitigation plan if known. If it is not yet known, no problem. -->

**Dependency: Flux, ArgoCD, etc.** - These GitOps tools/platforms must be able to detect the Radius definition files specified in the git repo and update the relevant Kubernetes clusters accordingly to deploy and manage Radius resources.

**Risk: ability for GitOps tools to deploy and manage Radius resources** - The primary risk is whether Flux, ArgoCD, etc. can deploy and manage Radius resources as specified in the git repo. We will need to test this to ensure that it works as expected.

## Key assumptions to test and questions to answer
<!-- If you are making assumptions that, if incorrect, would cause us to significantly alter our approach to this scenario, make them explicit here.  Also call out how / when you plan to validate key assumptions. -->
<!-- What big questions must we answer in order to clarify our plan for this scenario.  When and how do you plan to answer those questions (prototype feature x, customer research, competitive research, etc) -->
**Assumption:** GitOps tools (beginning with Flux and ArgoCD) must be able to detect the file change in the git repo described above and must be able to read the Bicep files as required to deploy the Radius app correctly. We will need to test this to ensure that it works as expected.

# Scenario 3: Patch and edit the configuration in my Kubernetes cluster through my GitOps toolset

## Target customers
<!-- Of the customers / personas listed in the Epic doc, what subset are we delivering this scenario to serve? -->
**Site Reliability Engineers (SREs)** - They leverage GitOps by checking in configuration files into their git repositories that add information for the customization of the workload, i.e. replica pod number, strategic merge solutions.

## Existing customer problem
<!-- <Write this in first person. You basically want to summarize what “I” as a customer am trying to accomplish, why the current experience is a problem and the impact it has on me, my team, my work and or biz, etc…. i.e. “When I try to do x aspect of cloud native app development, I have the following challenges / issues….<details>. Those issues result in <negative impact those challenges / issues have on your work and or business.> -->
As an SRE, I am expected to make adjustments to the configuration of the Kubernetes cluster and the Radius Environment as needed. Today, I adjust the cluster as needed through my GitOps toolset, but I'm unable to do so for all the Radius-specific resources since Radius types and Bicep files are not currently readable by Flux or ArgoCD. This means that I have to manually manage the Radius instance I am providing for my developers and operators to use, which is a significant overhead for me to adopt and to provide value to my development counterparts

## Desired customer experience outcome
<!-- <Write this as an “I statement” that expresses the new capability from customer perspective … i.e. After this scenario is implemented “I can do, x, y, z, steps in cloud native app developer and seamlessly blah blah blah …. As a result <summarize positive impact on your work / business>  -->
As an example, the application team will require 10 replica pods for their prod environment however this variable is currently set to 1. To fix this as a SRE, I should have 2 options:

**Option 1**

1. Using my editor of choice, VSCode, I'll make changes to the replica number for the pods through a YAML patch file that can contain the path to parameter annotations that effect the replica number of pods:

Example:

```yaml
apiVersion: kustomize...
kind: Component
resources:
- backend/mongoDB.yaml
patches:
- target:
    group: core.opera.prod
    version: 1
    kind: Application
    name: backend
  path: backend/replica-patch.yaml
```

2. This file points to replica-patch.yaml which contains parameters to change the replica number of pods to 10.
2. These 2 files are then committed to my configurational files to patch my Kubernetes cluster.

> As an SRE, I'm very happy with this experience because I did not have to change anything in my normal workflow behavior and was able to leverage Flux and my normal YAML file workflow.

**Option 2 Advanced SRE**

1. Using my editor of choice, VSCode, I'll make changes to the replica number for the pods through changing the parameter of replica sets that my MongoDB recipe takes in as defined in its Radius Environment resource in `env.bicep`:

```bicep
resource env 'Applications.Core/environments@2023-10-01-preview' = {
    name: 'prod'
    properties: {
      compute: {
        kind: 'kubernetes'
        resourceId: 'self'
        namespace: 'default'
      }
      recipes: {
        'Applications.Datastores/mongoDB':{
          'mongoDB-bicep': {
            templateKind: 'bicep'
            templatePath: 'https://ghcr.io/USERNAME/recipes/myrecipe:1.1.0'
            // Optionally set parameters for all resources calling this Recipe
            parameters: {
              replicaSet: 10
            }
          }
        }
      }
    }
}
```

2. I can then commit this file to my repository and have Flux apply it to my Radius Environment.

> As an SRE, I'm very happy with this experience because I did not have to do many steps to apply my parameter patch and while I had to learn some Bicep knowledge and Radius knowledge, I was able to ultimately leverage Flux mechanisms which I’m familiar with to apply my patches to the Kubernetes cluster.

_Requirements resulting from this scenario:_
- Flux must be able to detect the file change described above and must be able to read the env.bicep file as required to deploy the Radius app correctly 
- Radius types and Bicep files must be able to be read by Flux which currently does not have that ability.

## Key investments
<!-- List the features required to enable this scenario. -->

### Feature 1
<!-- One or two sentence summary -->

### Feature 2
<!-- One or two sentence summary -->

### Feature 3
<!-- One or two sentence summary -->

## Key dependencies and risks
<!-- What dependencies must we take in order to enable this scenario? -->
<!-- What other risks are you aware of that need to be mitigated. If you have a mitigation in mind, summarize here. -->
<!-- Dependency Name – summary of dependency.  Issues/concerns/risks with this dependency -->
<!-- Risk Name – summary of risk.  Mitigation plan if known. If it is not yet known, no problem. -->

## Key assumptions to test and questions to answer
<!-- If you are making assumptions that, if incorrect, would cause us to significantly alter our approach to this scenario, make them explicit here.  Also call out how / when you plan to validate key assumptions. -->
<!-- What big questions must we answer in order to clarify our plan for this scenario.  When and how do you plan to answer those questions (prototype feature x, customer research, competitive research, etc) -->

## Current state
<!-- If we already have some ongoing investment in this area, summarize the current state and point to any relevant documents. -->