# Title

* **Status**: Pending
* **Author**: Yetkin Timocin (@ytimocin)

## Overview

Radius CLI has multiple interactive commands that provide very important value to its users. Because these commands are very important for the functionality of Radius, we must make sure that we are able to cover different scenarios of these commands end to end. This design document will be about adding a tool to add functional tests for interactive Radius commands.

## Terms and definitions

Interactive Command: A command that needs to get user input. For example `rad init`.
Pseudo-terminal: A pseudo-terminal is a software emulation of physical terminal that can be used to control terminal applications programmatically. We are going to be using pty and vt10x.

## Objectives

> **Issue Reference:** <https://github.com/radius-project/radius/issues/6572>

### Goals

* Add a tooling so that adding functional/e2e tests for interactive Radius commands is very easy for all the contributors.
* Add a tooling so that adding functional/e2e tests for different paths of a given interactive Radius command is very easy for all the contributors.
* Add functional/e2e tests for the interactive commands so that we know if something is broken.

### Non goals

### User scenarios (optional)

#### User story 1

We have had two problems with the installation of dev recipes during the `rad init` command run. Right after releasing Radius, we noticed that dev recipes were not being installed by the `rad init` command. This could have been easily caught if we had had a functional test for the `rad init` command in place.

#### User story 2

A Radius contributor would like to add a new command that requires user input which means that the command is interactive. The contributor finishes writing the code and would like to add an e2e test. At that time, the contributor can use the new interactive command test tool to write an e2e test.

In the future, another contributor would like to change the flow of the interactive command that was mentioned above. The change breaks one of the e2e tests which is very good because it is caught at the pull request level.

## Design

* <https://github.com/Netflix/go-expect>: "An expect-like golang library to automate control of terminal or console based programs."
* <https://github.com/hinshun/vt10x>: "About Package vt10x is a vt10x terminal emulation backend."
* <https://github.com/creack/pty>: "PTY (PseudoTeletYpe) interface for Go."

### Design details

<!--
This section should be detailed and thorough enough that another developer
could implement your design and provide enough detail to get a high confidence
estimate of the cost to implement the feature but isn’t as detailed as the 
code. Be sure to also consider testability in your design.

For each change, give each "change" in the proposal its own section and
describe it in enough detail that someone else could implement it. Cover
ALL of the important decisions like names. Your goal is to get an agreement
to proceed with coding and PRs.

If there are alternatives you are considering please include that in the open
questions section. If the product has a layered architecture, it's good to
align these sections with the product's layers. This will help readers use
their current understanding to understand your ideas.

* Advantages of this design - Describe what's good about this plan relative to
  other options. Does it feel easy to implement? Provide flexibility for
  future work?
* Disadvantages - Describe what's not ideal about this plan. If you don't
  point these things out other people will do it for you. This is a good place
  to cover risks.
-->

### API design (if applicable)

* New structs to be added: Prompt, and InteractiveCommandRunner.
* A Prompt will have Command Arguments array and Procedure which is an array of interactive actions. Procedure will mostly consist of expectations with the help of Netflix's `go-expect` library.
* An InteractiveCommandRunner will have Run method which will take a Prompt.

## Alternatives considered

<!--
Describe the alternative designs that were considered or should be considered.
Give a justification for why alternative approaches should be rejected if
possible. 
-->

## Test plan

<!--
Include the test plan to validate the features including the areas that
need functional tests.

Describe any functionality that will create new testing challenges:
- New dependencies
- External assets that tests need to access
- Features that do I/O or change OS state and are thus hard to unit test
-->

## Security

* Some tests will need access to the account ids and/or secrets of, for example, AWS and Azure. At that moment, we are going to get these variables/secrets from the repository context which means that there will be no addition of these variables to the code therefore to the Radius repository.

## Compatibility (optional)

If there is a breaking change to one of the interactive commands we have, the tests will be broken which is basically what we want out of this tool. Instead of catching the problem in the production after the release, we would like to catch it in the pull request level.

## Monitoring

This is an addition to the test tools we have so that we probably don't need to send metrics from this tool that will have been created at the end of this design process and development.

With that being said, I am not sure if we should think about adding a way to identify flaky tests with some metrics. We can discuss this further.

## Development plan

* Creating the prototype (First PR).
* Cleaning up and making the prototype a tool that can be used for all commands (Second PR).
* Adding tests for different scenarios of `rad init` (Third PR).
* Creating issues on GitHub for some of the other interactive commands we have that should be covered.

## Open issues

* Create a functional test for `rad init`: <https://github.com/radius-project/radius/issues/6572>

## References

* <https://www.baeldung.com/linux/pty-vs-tty>
