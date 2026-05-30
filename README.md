# devops-databricks-asset-bundles

This repository contains reusable Azure DevOps pipeline templates for deploying Databricks Declarative Automation Bundles, formerly Databricks Asset Bundles. The templates are designed to standardize and streamline the CI/CD process across multiple projects.

## Repository Structure
```
devops-databricks-asset-bundles/
├── databricks-bundle-pipeline-template.yml       # Main Databricks bundle pipeline template
├── steps/
│   ├── checkout-self.yml                         # Step template for repository checkout
│   ├── configure-databricks-cli.yml              # Step template for Databricks CLI configuration
│   ├── deploy-databricks-bundle.yml              # Step template for deploying Databricks bundles
│   ├── install-databricks-cli.yml                # Step template for installing Databricks CLI
│   ├── run-databricks-jobs.yml                   # Step template for running Databricks jobs
│   ├── run-unit-tests.yml                        # Step template for running unit tests
│   ├── setup-python.yml                          # Step template for Python setup and dependencies
│   └── validate-databricks-bundle.yml            # Step template for bundle validation
└── README.md
```

## Templates Overview

### databricks-bundle-pipeline-template.yml

This is the main pipeline template for Databricks bundle deployments. It is modularized using step templates for setup, validation, deployment, and staging job execution. The template supports validation on all branches except `main` and `dev`, and runs validation as part of the release process for `dev` and `main` to avoid redundant setup.

### Step Templates (in `steps/`)
- **checkout-self.yml**: Checks out the repository with credentials and cleans the workspace.
- **setup-python.yml**: Installs Python and required dependencies including pytest.
- **install-databricks-cli.yml**: Installs the Databricks CLI.
- **configure-databricks-cli.yml**: Configures the Databricks CLI with environment variables.
- **run-unit-tests.yml**: Dynamically discovers and runs pytest unit tests from the `tests/` directory.
- **validate-databricks-bundle.yml**: Runs Databricks bundle validation.
- **deploy-databricks-bundle.yml**: Handles deployment of Databricks bundles to the specified environment.
- **run-databricks-jobs.yml**: Runs Databricks jobs as specified in the pipeline parameters.

## Prerequisites

These templates are built for **Azure DevOps Pipelines** and use Azure DevOps-specific features (pipeline `extends`, variable groups, Azure CLI tasks, etc.). They are not compatible with GitHub Actions.

To use this repository:

1. **Fork or import** this repository into your Azure DevOps organization. In Azure DevOps, go to **Repos > Import a repository** and use the clone URL: `https://github.com/brianjmurray/devops-databricks-asset-bundles.git`
2. **Reference the imported repo** as a resource in your project pipelines (see example below).

## Usage

To use these templates in your projects, follow these steps:

1. **Add the imported repository as a resource** in your project pipeline.
2. **Reference the main template** in your project's `azure-pipelines.yml` file.

### Example Project Pipeline

For a Databricks project, create an `azure-pipelines.yml` file in your project repository and reference the template as follows:

```yaml
trigger:
  branches:
    include:
      - main

pr:
  branches:
    include:
      - main

resources:
  repositories:
    - repository: templates
      type: git
      name: <your-azure-devops-project>/devops-databricks-asset-bundles

extends:
  template: databricks-bundle-pipeline-template.yml@templates
  parameters:
    projectName: 'YourProjectName'
    workingDirectory: 'YourWorkingDirectory'
    azureSubscription: 'yourAzureServiceConnection'
    jobNames: ['Job1', 'Job2']
    devVariableGroup: 'yourDevVariableGroup'
    stagingVariableGroup: 'yourStagingVariableGroup'
    prodVariableGroup: 'yourProdVariableGroup'
```

**Parameters:**
- `projectName`: Optional project label retained for consuming pipeline compatibility. It is not used by the template logic.
- `workingDirectory`: Path to the directory containing the Databricks bundle. In a monorepo with multiple bundles, set this to the specific bundle directory; all validate, deploy, run, and test commands execute from this path.
- `azureSubscription`: Name of the Azure DevOps service connection used to authenticate with Azure and obtain a Databricks access token.
- `jobNames`: List of Databricks jobs to run after staging deployments. Production deploys do not run jobs by default.
- `devVariableGroup`: Variable group for the development environment.
- `stagingVariableGroup`: Variable group for the staging environment.
- `prodVariableGroup`: Variable group for the production environment.
- `autoApprove`: (Optional) Boolean to automatically approve destructive changes during deployment. When set to `true`, adds the `--auto-approve` flag to `databricks bundle deploy` commands, bypassing manual approval prompts for operations that might delete or modify existing resources. Default is `false` for safety. **Use with caution in production environments**.
- `waitForJobs`: (Optional) Boolean to wait for Databricks jobs to complete after deployment. When set to `false`, jobs are triggered with `--no-wait` flag and the pipeline continues without waiting. Default is `true`. Set to `false` for long-running jobs (> 1 hour) to avoid pipeline timeouts.

## Unit Testing

The pipeline automatically discovers and runs unit tests from the `tests/` directory under your configured `workingDirectory`.

### How It Works
- Tests are executed before bundle validation and deployment
- If `${workingDirectory}/tests/` exists with files matching `test_*.py` or `*_test.py`, pytest runs automatically
- If no tests are found, the pipeline continues without failing
- Test results and code coverage reports are published to Azure DevOps
- Both pytest-style and unittest-style tests are supported

### Setting Up Tests
1. Create a `tests/` directory under the directory passed as `workingDirectory`
2. Add test files following pytest naming conventions:
   - `test_*.py` or `*_test.py`
3. Write your tests using standard pytest or unittest syntax

### Example Test Structure
```
YourProject/
├── databricks/
│   ├── src/
│   ├── databricks.yml
│   └── tests/
│       ├── __init__.py
│       ├── test_transformations.py
│       └── test_utilities.py
```

### Test Requirements
The following testing packages are pre-installed:
- pytest (can run both pytest and unittest tests)
- pytest-cov (for code coverage)
- pyspark (for Spark testing)

### Viewing Test Results
- Test results appear in the Azure DevOps "Tests" tab
- Code coverage reports appear in the "Code Coverage" tab
- Failed tests will block the pipeline from proceeding to deployment

## Job Execution

On the `dev` branch staging stage, jobs specified in `jobNames` are executed after deployment and the pipeline waits for completion by default. The `main` branch production stage validates and deploys only; it does not run jobs unless you extend the template. For long-running staging jobs that exceed the pipeline timeout (typically 1 hour), set `waitForJobs: false` to trigger jobs without waiting.

**Example with long-running jobs:**
```yaml
extends:
  template: databricks-bundle-pipeline-template.yml@templates
  parameters:
    projectName: 'MyProject'
    jobNames: ['LongETLJob', 'HourlyAggregation']
    waitForJobs: false  # Trigger but don't wait
    # ... other parameters
```

**Note:** When `waitForJobs: false`, the pipeline will show success even if jobs fail later. Check job status in the Databricks UI.

## Deployment Options

### Auto-Approve Feature {#autoapprove}

The `autoApprove` parameter controls whether deployments require manual approval for destructive changes. When enabled, it adds the `--auto-approve` flag to Databricks bundle deployment commands.

**When to use `autoApprove: true`:**
- Automated CI/CD pipelines where manual intervention is not desired
- Development environments where data loss is acceptable
- Well-tested deployments with comprehensive rollback procedures

**When to keep `autoApprove: false` (default):**
- Production environments
- Deployments that might affect existing data or resources
- Initial deployments to new environments
- When you want to review changes before they're applied

**Example with auto-approve enabled:**
```yaml
extends:
  template: databricks-bundle-pipeline-template.yml@templates
  parameters:
    projectName: 'MyProject'
    workingDirectory: 'src'
    autoApprove: true  # Enable auto-approval for this deployment
    # ... other parameters
```

**Note:**
- The pipeline is designed so that validation runs for all branches except `main` and `dev`. For `dev` and `main`, validation is included in the release process to avoid redundant setup steps and speed up deployments.
- When passing environment-specific parameters (like `env`), use the variable from your variable group, e.g.:
  ```yaml
  env: $(env)
  ```

## Authentication and Secrets

The template does not require storing Databricks personal access tokens in Azure Key Vault or Azure DevOps variable groups. `configure-databricks-cli.yml` obtains a short-lived Microsoft Entra access token at runtime with Azure CLI using the Azure DevOps service connection.

Use Azure Key Vault or secured variable groups for application secrets and environment-specific configuration values, such as workspace host names or downstream service credentials. The pipeline identity still needs the required Databricks workspace permissions for the bundle resources it deploys.

## Contributing

We welcome contributions to improve and expand the templates. Please submit a pull request with your changes and ensure that all existing tests pass.

## Contact

For any questions or support, please open an issue at [github.com/brianjmurray/devops-databricks-asset-bundles](https://github.com/brianjmurray/devops-databricks-asset-bundles/issues).
