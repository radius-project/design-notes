# Pull Request Merge Guide

This guide provides comprehensive instructions for merging pull requests (PRs) in the radius-project repositories, with specific focus on the design-notes repository process.

## Table of Contents

1. [General GitHub PR Merge Process](#general-github-pr-merge-process)
2. [Design Notes Repository PR Process](#design-notes-repository-pr-process)
3. [Git and GitHub CLI Commands](#git-and-github-cli-commands)
4. [Testing PRs on Forks](#testing-prs-on-forks)
5. [Best Practices](#best-practices)

## General GitHub PR Merge Process

### Prerequisites for Merging
- PR has been reviewed and approved by required reviewers
- All required status checks are passing
- Any merge conflicts have been resolved
- Branch is up-to-date with the target branch

### Merge Methods

GitHub provides three merge methods:

1. **Merge Commit**: Creates a merge commit that combines the feature branch with the base branch
2. **Squash and Merge**: Squashes all commits from the feature branch into a single commit
3. **Rebase and Merge**: Replays the commits from the feature branch onto the base branch without creating a merge commit

### Steps to Merge

#### Via GitHub Web Interface
1. Navigate to the pull request
2. Ensure all checks are passing and approvals are in place
3. Click the "Merge pull request" button
4. Choose your preferred merge method
5. Add a commit message if needed
6. Click "Confirm merge"
7. Delete the feature branch if no longer needed

#### Via GitHub CLI
```bash
# View PR status
gh pr status

# View specific PR details
gh pr view [PR_NUMBER]

# Merge PR with default method
gh pr merge [PR_NUMBER]

# Merge with specific method
gh pr merge [PR_NUMBER] --merge     # Creates merge commit
gh pr merge [PR_NUMBER] --squash    # Squash and merge
gh pr merge [PR_NUMBER] --rebase    # Rebase and merge

# Merge with custom commit message
gh pr merge [PR_NUMBER] --squash --body "Custom merge message"
```

## Design Notes Repository PR Process

The design-notes repository follows a specific review and merge process for design documents:

### Before Merging

1. **Design Review Complete**: The design has been reviewed in a design review meeting (for major topics) or asynchronously approved (for minor topics)

2. **Feedback Incorporated**: All outstanding feedback from reviewers has been addressed

3. **Comments Resolved**: All comments in the PR have been responded to and resolved, with clear explanations of how feedback was addressed

4. **Approvals Obtained**: One or more approvers have approved the pull request

### Merge Process

1. **Final Review**: Ensure all feedback is incorporated and documented
2. **Resolve Comments**: Respond to all comments explaining resolution
3. **Get Approvals**: Wait for required approvals from maintainers
4. **Merge**: Once approved, merge the pull request
5. **Implementation**: Development work on implementing the design can begin only after merge

### Post-Merge Updates

If updates arise during implementation that need review:
- Create a new PR with proposed design updates
- Follow the same design review process for the updates

## Git and GitHub CLI Commands

### Checking Out PRs to Your Fork

These commands address the open question in the workflow changes document about how reviewers can test PRs on their own forks:

#### Method 1: Using GitHub CLI
```bash
# Clone the original repository to your fork
gh repo fork radius-project/[REPO_NAME] --clone

# Checkout a specific PR to test
gh pr checkout [PR_NUMBER]

# Push to your fork for testing
git push origin [BRANCH_NAME]
```

#### Method 2: Using Git Commands
```bash
# Add the original repository as upstream (if not already added)
git remote add upstream https://github.com/radius-project/[REPO_NAME].git

# Fetch the PR branch
git fetch upstream pull/[PR_NUMBER]/head:[LOCAL_BRANCH_NAME]

# Checkout the PR branch
git checkout [LOCAL_BRANCH_NAME]

# Push to your fork for testing
git push origin [LOCAL_BRANCH_NAME]
```

#### Method 3: Direct PR Branch Checkout
```bash
# Fetch PR branch directly
git fetch origin pull/[PR_NUMBER]/head:pr-[PR_NUMBER]

# Checkout the PR branch
git checkout pr-[PR_NUMBER]

# Push to your fork
git remote add myfork https://github.com/[YOUR_USERNAME]/[REPO_NAME].git
git push myfork pr-[PR_NUMBER]
```

### Testing Workflow Changes on Forks

For repositories with GitHub workflow changes:

```bash
# Checkout PR to your fork
gh pr checkout [PR_NUMBER]

# Push to your fork's branch
git push origin [BRANCH_NAME]

# Manually trigger workflows on your fork
gh workflow run [WORKFLOW_NAME] --repo [YOUR_USERNAME]/[REPO_NAME]

# Check workflow status
gh run list --repo [YOUR_USERNAME]/[REPO_NAME]
```

## Testing PRs on Forks

### When to Test on Forks
- PRs containing GitHub workflow modifications
- Changes to automation scripts or build processes
- New features that need validation in an isolated environment

### Fork Testing Process

1. **Fork Setup**:
   ```bash
   # Fork the repository
   gh repo fork radius-project/[REPO_NAME] --clone
   
   # Configure upstream
   git remote add upstream https://github.com/radius-project/[REPO_NAME].git
   ```

2. **PR Checkout**:
   ```bash
   # Fetch and checkout PR
   gh pr checkout [PR_NUMBER]
   
   # Push to your fork
   git push origin [BRANCH_NAME]
   ```

3. **Test Changes**:
   ```bash
   # For workflow changes - manually trigger
   gh workflow run [WORKFLOW_NAME] --repo [YOUR_USERNAME]/[REPO_NAME]
   
   # For code changes - run local tests
   make test  # or appropriate test command
   ```

4. **Provide Feedback**:
   - Comment on the original PR with test results
   - Include links to workflow runs on your fork
   - Report any issues discovered during testing

## Best Practices

### For PR Authors

1. **Clear Description**: Provide clear PR description explaining changes and their purpose
2. **Small, Focused Changes**: Keep PRs focused on a single concern when possible
3. **Tests Included**: Include tests for new functionality
4. **Documentation Updated**: Update relevant documentation
5. **Self-Review**: Review your own PR before requesting reviews

### For PR Reviewers

1. **Thorough Review**: Review code, tests, and documentation
2. **Test on Fork**: For workflow changes, test on your own fork
3. **Constructive Feedback**: Provide actionable, helpful feedback
4. **Timely Response**: Respond to PR reviews in a timely manner

### For Maintainers/Mergers

1. **Verify Approvals**: Ensure required approvals are in place
2. **Check Status**: Confirm all status checks are passing
3. **Review Changes**: Do a final review of changes before merging
4. **Choose Appropriate Merge Method**: 
   - Use "Squash and merge" for feature branches with multiple small commits
   - Use "Merge commit" to preserve commit history for complex features
   - Use "Rebase and merge" for clean, linear history

### Workflow-Specific Guidelines

For repositories following the workflow design principles:

1. **Testability**: Ensure workflow changes can be tested on developer machines
2. **Fork Compatibility**: Verify workflows can run on forks using `workflow_dispatch`
3. **No Fork Schedules**: Confirm scheduled workflows don't trigger on forks
4. **Make Integration**: Verify core logic is accessible via Make commands
5. **Environment Variables**: Ensure configuration uses environment variables

### Troubleshooting Common Issues

#### Merge Conflicts
```bash
# Sync with upstream
git fetch upstream main
git checkout [YOUR_BRANCH]
git merge upstream/main

# Resolve conflicts and commit
git add .
git commit -m "Resolve merge conflicts"
git push origin [YOUR_BRANCH]
```

#### Failed Status Checks
- Review the failing check details
- Fix issues locally and push updates
- Re-run checks if they were transient failures

#### Missing Approvals
- Ping reviewers if reviews are overdue
- Address any outstanding feedback
- Ensure all required reviewers have approved

## Additional Resources

- [GitHub Documentation on Merging PRs](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/incorporating-changes-from-a-pull-request/merging-a-pull-request)
- [GitHub CLI Documentation](https://cli.github.com/manual/)
- [Radius Contributing Guidelines](https://github.com/radius-project/radius/blob/main/CONTRIBUTING.md)
- [Radius Workflow Changes Design](./tools/2025-03-workflow-changes.md)