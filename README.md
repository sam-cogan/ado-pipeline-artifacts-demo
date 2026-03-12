# Azure DevOps Pipeline Artifacts Demo

This repository demonstrates how to correctly consume artifacts from the same branch in Azure DevOps pipelines.

## Overview

This demo includes three pipelines that work together to show two different patterns for branch-aware artifact consumption:

1. **build-pipeline.yml** - Builds and publishes artifacts for each branch
2. **deploy-pipeline-triggered.yml** - Automatically deploys when build completes (Option A)
3. **deploy-pipeline-manual.yml** - Manually deploy from a specific branch (Option B)

## Pipeline Descriptions

### 1. Build Pipeline (`build-pipeline.yml`)

**Purpose**: Builds the application and publishes artifacts for each branch.

**Key Features**:
- Triggers on all branches (`*`)
- Creates a `build-info.txt` file containing:
  - Branch name
  - Build ID
  - Build number
  - Commit hash
  - Timestamp
- Publishes artifact named `drop`

**How to Use**:
1. Import this pipeline in Azure DevOps
2. Push to any branch to trigger a build
3. Each build creates a unique artifact tagged with the build ID

### 2. Deploy Pipeline - Triggered (Option A)

**Purpose**: Automatically deploy when the build pipeline completes on any branch.

**Key Features**:
- Uses `resources.pipelines` to reference the build pipeline
- Auto-triggers when build completes
- Uses `$(resources.pipeline.myBuildPipeline.runID)` to download the exact artifact from the triggering build
- **Guarantees** branch consistency - always deploys the artifact from the build that triggered it

**How to Use**:
1. Import this pipeline in Azure DevOps
2. Update the `source:` field to match your build pipeline name (if different)
3. When the build pipeline completes, this pipeline automatically runs
4. Review the output to see the artifact branch information

**When to Use This Pattern**:
- CI/CD scenarios where you want automatic deployment after build
- When you need to deploy the exact build that just completed
- When you want to maintain strict branch consistency between build and deploy

### 3. Deploy Pipeline - Manual (Option B)

**Purpose**: Manually deploy the latest artifact from the same branch the pipeline is running on.

**Key Features**:
- Manual trigger only
- Auto-detects the current branch using `$(Build.SourceBranchName)`
- Uses `buildVersionToDownload: latestFromBranch` to get the latest successful build from the current branch
- Includes validation to verify the artifact matches the current branch

**How to Use**:
1. Import this pipeline in Azure DevOps
2. Switch to the branch you want to deploy (e.g., `git checkout develop`)
3. Manually trigger the pipeline from that branch in Azure DevOps
4. The pipeline automatically downloads the latest artifact from the branch it's running on

**When to Use This Pattern**:
- When you want to deploy based on the pipeline's current branch context
- Manual deployment scenarios where the branch is implicit
- When you want to deploy the latest "known good" build from the current branch
- For branch-specific deployments without needing to specify the branch name

## Key Concepts Demonstrated

### Branch-Correct Artifact Consumption

Both deploy pipelines ensure you get artifacts from the correct branch, but in different ways:

**Option A (Triggered)**:
```yaml
runId: '$(resources.pipeline.myBuildPipeline.runID)'
```
- Downloads from the specific build run that triggered the deployment
- Automatic branch consistency
- Best for continuous deployment

**Option B (Manual)**:
```yaml
buildVersionToDownload: 'latestFromBranch'
branchName: '$(Build.SourceBranch)'
```
- Downloads from the latest successful build on the current branch
- Auto-detects the branch the pipeline is running on
- Best for branch-aware manual deployments

### Artifact Verification

Each artifact includes a `build-info.txt` file that clearly shows:
- Which branch it was built from
- The build ID and number
- When it was built

This makes it easy to verify that you're deploying the correct artifact.

## Setup Instructions

### 1. Create the Build Pipeline

1. In Azure DevOps, go to **Pipelines** → **New pipeline**
2. Select your repository
3. Choose "Existing Azure Pipelines YAML file"
4. Select `build-pipeline.yml`
5. Save and run the pipeline
6. **Important**: Note the pipeline name (it will be used in deploy pipelines)

### 2. Create Deploy Pipeline - Triggered

1. Create a new pipeline
2. Select `deploy-pipeline-triggered.yml`
3. **Update line 16** if needed: Change `source: 'build-pipeline'` to match your build pipeline name
4. Save the pipeline (don't run it yet)
5. The pipeline will automatically trigger when the build pipeline completes

### 3. Create Deploy Pipeline - Manual

1. Create a new pipeline
2. Select `deploy-pipeline-manual.yml`
3. **Update line 43** if needed: Change `definition: 'build-pipeline'` to match your build pipeline name
4. Save the pipeline
5. Run manually and specify a branch name

## Testing the Demo

### Test Automatic Deployment (Option A)

1. Create a new branch: `git checkout -b feature/test-branch`
2. Make a small change and push: `git push origin feature/test-branch`
3. The build pipeline will trigger and complete
4. The triggered deploy pipeline will automatically run
5. Check the deploy pipeline logs to verify it deployed the artifact from `feature/test-branch`

### Test Manual Deployment (Option B)

1. Ensure you have successful builds on multiple branches (e.g., `main` and `develop`)
2. In Azure DevOps, navigate to the manual deploy pipeline
3. Switch the branch selector to the branch you want to deploy (e.g., `develop`)
4. Click "Run pipeline" (no parameters needed)
5. Check the logs to verify:
   - The current branch is detected correctly
   - The artifact was downloaded from that same branch
   - The validation step confirms branch names match

### Test Branch Isolation

1. Build from two different branches (e.g., `main` and `feature/test`)
2. Run the manual deploy pipeline from the `main` branch - verify the artifact shows "Branch Name: main"
3. Run the manual deploy pipeline from the `feature/test` branch - verify the artifact shows "Branch Name: feature/test"
4. This confirms artifacts are correctly isolated by branch and the pipeline auto-detects the correct branch

## Common Issues and Solutions

### Issue: "Pipeline not found" error in deploy pipelines

**Solution**: Update the pipeline name in the deploy pipeline YAML:
- In `deploy-pipeline-triggered.yml`: Update line 16 (`source: 'build-pipeline'`)
- In `deploy-pipeline-manual.yml`: Update line 43 (`definition: 'build-pipeline'`)

### Issue: Triggered pipeline doesn't run automatically

**Solution**: 
1. Ensure the pipeline resource trigger is configured correctly
2. Check that the pipeline has run at least once manually
3. Verify you have permissions to trigger pipelines

### Issue: Manual pipeline deploys from wrong branch

**Solution**:
1. Ensure you're running the pipeline from the correct branch in Azure DevOps
2. Check the branch selector in the "Run pipeline" dialog
3. The pipeline uses the branch context it's executed from, not the default branch

### Issue: "Artifact not found" error

**Solution**:
1. Ensure the build pipeline has completed successfully
2. Verify the artifact name matches in both pipelines (`drop`)
3. Check that the build produced and published the artifact

## Best Practices

1. **Use Option A (Triggered) for CI/CD**: When you want automatic deployment after every successful build
2. **Use Option B (Manual) for Promotions**: When you need to deploy specific branches to specific environments
3. **Always Validate**: Include steps to verify the artifact matches the expected branch
4. **Use Descriptive Build Info**: Include branch, build ID, and timestamp in artifacts for easy verification
5. **Use Full Ref Paths**: When specifying branches, use `refs/heads/branch-name` format

## References

- [Azure DevOps Pipeline Artifacts](https://learn.microsoft.com/azure/devops/pipelines/artifacts/pipeline-artifacts)
- [Download Pipeline Artifacts Task](https://learn.microsoft.com/azure/devops/pipelines/tasks/reference/download-pipeline-artifact-v2)
- [Pipeline Resources](https://learn.microsoft.com/azure/devops/pipelines/process/resources)
