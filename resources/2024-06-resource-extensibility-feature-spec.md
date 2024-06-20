# Resource Extensibility (User Defined Resource Types)

* **Status**: Pending
* **Author**: Reshma abdul Rahim (@reshrahim)

## Summary
The cloud native landscape is changing and expanding at a rapid pace like never before. Many enterprises use a wide range of technologies together for achieving their cloud-native strategy.  For any technology that’s a newcomer and helps solve a problem in the cloud native landscape, users look for an easy and seamless way to integrate their existing tools and technologies and incrementally adopt the new technology to their strategy. For Radius, we have heard requests from our customers/community to support technologies that their applications are tightly coupled with E.g.: an internal messaging service or a technology they absolutely love E.g.: PostgreSQL/Kafka.

We need to enable the open-source community to build and experiment with imperfect things. We need to provide an extensibility model that supports “Bring your own technology”, define and use it with Radius. This will help us to meet the community where they are and enable them to experiment and leverage their work as open-source contributions.

One of the high value extensibility points in Radius is Recipes. We have received interests to create custom resource types, define Recipes for the custom resource types and use it in the Radius application. Today Radius Extenders helps in creating custom resource types, but they are untyped and have limitations. The goal of providing resource extensibility is to empower developers or infrastructure operators to author and run their applications with custom resource types seamlessly in Radius and use all the other features such as Recipes, connections and app graph with ease and flexibility.

### Top level goals
1. Enable users to author their applications with custom resource types in Radius without having to write Go code to integrate with Radius components and deploy their application seamlessly.
2. Automatically enable Radius features such as Recipes, Connections, App graph, rad CLI for custom resource types.
3. Enable users to contribute and open-source the custom resource types and recipes to the community. This will accelerate the adoption of Radius across the cloud native community.
4. Radius uses the resource extensibility framework to  refactor its already built in resource types.

### Non-goals (out of scope)



## Customer profile and challenges
**Enterprises**: Platform engineering teams or operations in enterprises focus on streamlining the developer-experience for their organization by defining a set of recommended practices for application teams and provide self-service capabilities for application teams/developers to use. Radius aids the platform engineering efforts with the help of Recipes where the platform engineers or operators define the infrastructure-as-code templates to create resources on demand when application is deployed. One of the major challenges that exists with the Recipes today is the limited number of resources supported. Radius doesn’t provide an extensibility model for users to bring their custom resources, define Recipes and deploy them.

**Open-source community**: Building a sustainable open-source community is crucial to the success of any open-source project, including Radius. We need to cultivate an ecosystem for the open-source community to thrive, innovate and produce high quality work. Users from the community are motivated to contribute for different reasons:
    -  They want to use the project at their work and need a feature, 
    -  The project triggered their interests in the cloud native landscape and want to use the technology.
    -  They want to code in a particular language E.g. :Go
    -  They want to integrate their existing tools and technologies with the project

Today Radius enables users to get started on contributing to Radius with good-first-issues but doesn’t have a model to promote contributions further. Since the beginning of its open source launch, Dapr created the components repository and invited users to build their own components to unblock their scenarios and in turn folks contributed to the project. Learning from Dapr, Radius needs to have the extensibility points defined for contributors to interact with the project so that they can build out the things that they need to enable their scenarios and in turn the community gets benefitted with those contributions.

### Customer persona(s)
<!-- Who is the target customer? Include size/org-structure/decision makers where applicable. -->
- Platform engineers: Platform engineers are responsible for streamlining the developer-experience for their organization by defining a set of recommended practices for application teams and provide self-service capabilities for application teams/developers to use. They are responsible for building the platform and providing the necessary tools and services for the application teams to deploy their applications.

- IT Operators: IT operators are responsible for managing the infrastructure and ensuring that the applications are running smoothly. They are responsible for maintaining the infrastructure and providing support for the infrastructure. They are the primary users of Radius Recipes as they are responsible for defining the infrastructure-as-code templates for the applications.

- System Reliability Engineers (SREs) : SREs are responsible for ensuring the reliability of the applications and services that are deployed in the cloud. They are responsible for maintaining the infrastructure and ensuring that the applications are running smoothly. They maintain the infrastructure-as-code templates and provide support for the applications that are deployed. 

- System integrators : System integrators are partners who help enterprises integrate proprietary services with cloud native applications. They are responsible for designing and building cloud native applications on behalf of their customers.

- Developers : Developers are responsible for building the cloud native applications. They are responsible for writing the code, designing and maintaining the applications.

- Open-source contributors : Cloud native open-source contributors can be any of the above personas who are interested in contributing to the cloud native projects. 

### Positive customer outcomes
- Author and deploy : I can bring my own applications with custom resources/services and integrate with Radius seamlessly. I don’t have to write Go code to integrate with Radius components and can use simple spec to generate the resource definition and integrate with Radius 

- Recipes for user defined types: I can create Recipes for the custom resource types and deploy via Radius 

- OSS contributions: I can create and contribute Recipes for the custom resource types for the community to use.

## Key scenarios

### Scenario 1: Deb integrates the Budgets app with Radius
Deb, a platform engineer at a Financial Services company, wants to integrate the Budgets app with Radius. The Budgets app relies on an internal messaging service called Plaid. Deb wants to use Plaid as a resource type in Radius to deploy the application seamlessly. He needs a way to define and use Plaid as a custom resource type in Radius without having to write Go code. This will enable him to leverage all the features of Radius, such as Recipes, Connections, and the rad CLI, with ease and flexibility.

### Scenario 2: Amy contributes and open-sources PostgreSQL support to Radius
Amy is a system integrator who helps customers build cloud native applications on AWS. Amy heard about Radius and starts integrating their customer workloads in Radius. She sees a lot of her scenarios involve PostgreSQL and finds Radius doesn’t support that yet. She wants to contribute the PostgreSQL support to Radius and open source it so anyone can use it

### Scenario 3: Raj publishes Recipes for proprietary services
Raj is a partner or a system integrator who helps enterprises integrate proprietary service for eg : Oracle database in their cloud native applications. Raj tries out Radius and wants to write Recipes to create an Oracle Database with a standard set of best practices and policies integrated. He works with a lot of customers who are wanting to use the Oracle Database recipe in their applications.

## Key dependencies and risks
<!-- What dependencies must we take in order to enable this scenario? -->

<!-- What other risks are you aware of that need to be mitigated. If you have a mitigation in mind, summarize here. -->
<!-- **Dependency Name** – summary of dependency.  Issues/concerns/risks with this dependency -->
<!-- **Risk Name** – summary of risk.  Mitigation plan if known. If it is not yet known, no problem. --> 

- Dependency: Bicep compiler merge support
    - We need to ensure that the ongoing Bicep compiler merge work is complete so that custom resource types can be dynamically generated for the user defined resource types.
    - Mitigation: Work closely with the Bicep team to ensure that the merge is completed on time and that the necessary features are available to enable custom resource types in Radius .

- Risk: Adoption and community engagement
    - There is a risk that users may not fully embrace the extensibility feature or contribute custom resource types to the community.
    - Mitigation: Actively engage with users, provide clear documentation and resources, and foster a supportive and inclusive community to encourage adoption and contributions.

- Risk: Compatibility and maintainability
    - As more custom resource types are added to Radius, there is a risk of compatibility issues and increased maintenance overhead.
    - Mitigation: Implement thorough testing and versioning strategies to ensure compatibility, and establish clear guidelines for maintaining custom resource types.

- Risk: Security and trust
    - Allowing users to contribute custom resource types introduces potential security risks and the need to establish trust in the community-contributed code.
    - Mitigation: Implement strict security measures, such as code reviews and vulnerability scanning, and establish a transparent review process for community contributions.

## Key assumptions to test and questions to answer
<!-- If you are making assumptions that, if incorrect, would cause us to significantly alter our approach to this scenario, make them explicit here.  Also call out how / when you plan to validate key assumptions. -->
<!-- What big questions must we answer in order to clarify our plan for this scenario.  When and how do you plan to answer those questions (prototype feature x, customer research, competitive research, etc) -->

- Assumption: Users will find value in the extensibility feature and actively contribute to the community.
- Assumption: Users will use extensibility in lieu of existing extenders because they want to take advantage of things like strong typing and validation in Radius

## Current state
<!-- If we already have some ongoing investment in this area, summarize the current state and point to any relevant documents. -->
[Radius Extenders](https://docs.radapp.io/guides/author-apps/custom/overview/#extenders) enables users to author custom resource types but they are weakly types. They allow you to pass in any property or secret and for cases where the developer/operator need to extend and reference a simple resource type definition. Extenders are for untyped/ weakly typed resources and do not provide a lot of customization for users wanting to have guardrails in their platforms with strongly typed resource definitions following a strict set of rules and restrictions. 

## Details of the customer problem
<!-- <Write this in first person. You basically want to summarize what “I” as a customer am trying to accomplish, why the current experience is a problem and the impact it has on me, my team, my work and or biz, etc…. i.e. “When I try to do x aspect of cloud native app development, I have the following challenges / issues….<details>. Those issues result in <negative impact those challenges / issues have on your work and or business.> -->

## Desired customer experience outcome
<!-- <Write this as an “I statement” that expresses the new capability from customer perspective … i.e. After this scenario is implemented “I can do, x, y, z, steps in cloud native app developer and seamlessly blah blah blah …. As a result <summarize positive impact on your work / business>  -->

### Detailed Customer Experience
 <!-- <List of steps the customer goes through from the start to the end of the scenario to provide more detailed view of exactly what the customer is able to do given the new capabilities>  -->
<!-- Step 1
Step 2
… -->

## Key investments
<!-- List the features required to enable this scenario. -->

### Feature 1
<!-- One or two sentence summary -->

### Feature 2
<!-- One or two sentence summary -->

### Feature 3
<!-- One or two sentence summary -->