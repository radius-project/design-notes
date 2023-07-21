# Design notes

This `design-notes` repository contains design proposals, enhancements, and architectural decisions for Radius. It provides a consistent and controlled path to record changes and developments, ensuring clarity and transparency for all stakeholders in the Radius community.

Minor changes such as documentation updates or small bug fixes may be reviewed and implemented directly via a GitHub issue. For larger changes, such as new feature design, a design process is required. These must get consensus from Radius maintainers and its community, and should follow the [design note template](./template/YYYY-MM-design-template.md) in this repository.

For more information on how to contribute to Radius visit [CONTRIBUTING.md](https://github.com/project-radius/radius/blob/main/CONTRIBUTING.md).

## Structure

The repository is organized into several sections, each corresponding to a specific area of Radius:

| Directory Name | Description |
|---|---|
| [architecture](./architecture/) | Contains the overall system architecture designs for Radius services. |
| [bicep](./bicep/) | Contains the designs related to deployment-engine, Bicep compiler, and Bicep types. |
| [cli](./cli/) | Contains the designs for Radius CLI. |
| [ucp](./ucp/) | Contains the designs for the Universal Control Plane (UCP). |
| [resources](./resources/) | Contains the designs for Radius resource types, such as those in the Application.* namespace. | 
| [recipe](./recipe/) | Contains the designs related to Radius recipes. |
| [tools](./tools/) | Contains the designs for engineering tools, such as GitHub Action Workflow and test-infra. |
| [template](./template/) | Contains the template for design documents.|

## Code of Conduct

Please refer to our [Radius Community Code of Conduct](https://github.com/project-radius/radius/blob/main/CODE_OF_CONDUCT.md)
