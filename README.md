# Design notes

This `design-notes` repository contains design proposals, enhancements, and architectural decisions for Radius. It provides a consistent and controlled path to record changes and developments, ensuring clarity and transparency for all stakeholders in the Radius community.

Design notes are used for evaluating the design of **planned** changes and additions to Radius. Do not use this repository for feature requests - please create an issue on the [main Radius repository](https://github.com/radius-project) instead. We create documents in this repo to describe **how** to accomplish something once we have agreement that it is appropriate for the project.

Minor changes such as documentation updates or small bug fixes may be reviewed and implemented directly via a GitHub issue. For larger changes, such as new feature design, a design note pull-request and review is required. These must get consensus from Radius maintainers and the community, and should follow the [design note template](./template/YYYY-MM-design-template.md) in this repository.

For more information on how to contribute to Radius visit [CONTRIBUTING.md](https://github.com/radius-project/radius/blob/main/CONTRIBUTING.md).

## Review note review process and instructions

Our design note review review process is based on pull-requests to this repository. 

- Major design topics will be reviewed during our design review meetings.
- Minor design review topics may be asynchronously reviewed without a meeting depending on capacity.

Our design review meeting is currently a **closed meeting**.

### Creating a pull-request 

Follow the following steps to create a pull-request:

- Create a new document using the [design note template](./template/YYYY-MM-design-template.md) in this repository.
  - Name the document using the year and month followed by a short description name. eg: `2023-04-tls-termination.md`
  - Place the document in the appropriate folder based on the area. See the `Structure` section in this readme for descriptions.
  - Fill out the template and **DO NOT** omit sections.
  - If you have supporting assets like images please place them a directory using the same name as the document (without `.md`).
- When you are ready for feedback:
  - Commit the document.
  - Push your changes.
  - Open a pull-request on this repository.

Please open your pull-request **before** the design meeting where you want to review the document. Community members that are subscribed to the repository will be notified when the pull-request is opened.

### During the review

During a review, community members and maintainers will ask questions and leave feedback. The design may be approved as-is or it may be returned for iteration.

As a reviewer: Please make a record of all major questions and feedback in the pull-request using comments. This allows the community to see the history and discussion even if they were not present when it was discussed.

### After the review: authors

If the design is approved:

  - Please incorporate any outstanding feedback.
  - Please respond to, and resolve all comments in the document explaining the resolution of the feedback. 
  - One or more approvers will approve the pull-request so that it can be merged.
  - Merge the pull-request.

Development work on implementing the design begins only after the design is approved.

If there are updates come up during development work that need consideration and team review, please create a pull-request with proposed updates to the design. The design review process will be repeated for this new pull-request.

If the design is not approved:

  - Please leave a comment on the pull-request explaining the major feedback and next steps for proceeding.
  - It is **your** responsibility as the author to document the next steps clearly so we can pick up where we left off.
  - Please iterate on the proposal and prepare the next review. This may include responding to and resolving feedback in the pull-request.

## Structure

The repository is organized into several sections, each corresponding to a specific area of Radius:

| Directory Name | Description |
|---|---|
| [architecture](./architecture/) | Contains the overall system architecture designs for Radius services. |
| [bicep](./bicep/) | Contains the designs related to deployment-engine, Bicep compiler, and Bicep types. |
| [cli](./cli/) | Contains the designs for Radius CLI. |
| [ucp](./ucp/) | Contains the designs for the Universal Control Plane (UCP). |
| [resources](./resources/) | Contains the designs for Radius resource types in `Application.*` namespace. | 
| [recipe](./recipe/) | Contains the designs related to Radius recipes. |
| [tools](./tools/) | Contains the designs for engineering tools, such as GitHub Action Workflow and test-infra. |
| [template](./template/) | Contains the template for design documents.|

## Code of Conduct

Please refer to our [Radius Community Code of Conduct](https://github.com/radius-project/radius/blob/main/CODE_OF_CONDUCT.md)
