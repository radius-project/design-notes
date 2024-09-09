# Title

- **Author**: Your name (@YourGitHubUserName)

## Overview

<!--
Provide a succinct high-level description of the system, feature, or component being analyzed. Explain why a threat model is being created for this system, feature, or component. This section should be one to three paragraphs long and understandable by someone outside the Radius team.
-->

## Terms and Definitions

<!--
Include any terms, definitions, or acronyms that are used in this threat model document to assist the reader. They may or may not be part of the user-facing experience once implemented, and can be specific to this design context.
-->

## System Description

<!--
Provide a detailed description of the system or feature being modeled. Include information key components, and interactions with other systems.
-->

### Architecture

<!-- Overview of the system architecture of the component that is being discussed in this document. -->

### Implementation Details

<!-- What are the components of the implementation -->

**Is there any use of cryptography?**

<!-- Answer YES/NO and, if yes, please describe (the type of the cryptography used, their purpose, and libraries used) -->

<!-- Examples can include encryption and hashing. -->

**Does the component store secrets?**

<!-- Answer YES/NO and, if yes, please describe the type of data and how it is stored. -->

**Does the component process untrusted data or does the component parse any custom formats?**

<!-- Answer YES/NO and, if yes, please describe the type of data and the libraries that are used to parse the data. -->

<!-- Ex: data coming from a user. -->

### Clients

<!-- Clients that communicate with the component that is being reviewed in the threat model. -->

## Trust Boundaries

## Data Flows

<!--
Include a diagram of the system architecture, showing how different components interact. Highlight any areas where security controls are implemented or where threats might be present.
-->

### Diagram

<!-- The diagram for the threat model. It can be done by using Microsoft Threat Modeling Tool. -->

## Threats

<!-- This is where we talk about data flows and its threats with detailed threats and mitigations. -->

### Threat 1: Threat about a component

**Description**
**Impact**
**Mitigation**
**Status**

## Open Questions

<!--
List any unresolved questions or uncertainties about the threat model. Use this section to gather feedback from experts or team members and to track decisions made during the review process.
-->

## Action Items

<!--
The list of action items that will be done in order to improve the safety of the system.
-->

## Review Notes

<!--
Update this section with the decisions and feedback from the threat model review meeting. Document any changes made to the model based on the review.
-->
