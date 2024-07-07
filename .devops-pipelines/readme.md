# Azure DevOps Slim CI for dbt

This repository contains Azure DevOps pipelines that implement a Continuous Integration (CI) workflow for dbt Core™ projects. It is an attempt to mimic the slim CI feature available when using dbt Cloud™, where the CI pipeline can be configured to reuse models from a production environment when building the dbt models.

The CI workflow is split into two main pipelines: `azure-pipelines-dbt-ci-main.yml` and `azure-pipelines-dbt-slim-ci.yml`. These pipelines automate the testing and deployment of dbt models, facilitating a streamlined development process for data transformation projects.

## Overview

The CI workflow consists of two pipelines:

1. **Main Branch Pipeline (`azure-pipelines-dbt-ci-main.yml`):** This pipeline is scheduled to run on the `main` branch, executing dbt commands to build all models contained in the production branch into a "ci_main" schema within the data warehouse. It is triggered at 6:30 AM UTC from Monday to Friday, ensuring that the latest changes are built into the ci_main schema at the start of the workday. The pipeline only runs if changes have been applied to the main branch since the last run.

2. **Pull Request Pipeline (`azure-pipelines-dbt-slim-ci.yml`):** This pipeline runs on each pull request (PR), using the "ci_main" schema to defer untouched models in the PR branch when evaluating the PR. This approach allows for a more efficient CI process, as only the models affected by the PR are rebuilt and tested, reducing the overall execution time considerably for medium or large projects.

<br>


### CI Workflow
The developer creates a feature branch and starts working on the dbt project, changing, adding dbt models as required. During development, the development data warehouse is used from the dbt Core tool (**1**) on the local development environment.
<br><br>
When the feature is complete the developer creates a pull request for review. This triggers a CI pipeline (**2**) that loads the dbt manifest generated from the main branch, and proceeds to build the dbt models deferring to the models on the ci_main schema. 
<br><br>
When the feature is merged to main this triggers a pipeline (**3**) that generates a dbt manifest from the main branch, and proceeds to build the dbt models to the ci_main schema. The dbt manifest is then uploaded to the storage account.
<br><br>
The main branch is deployed to the production data warehouse (**4**).

<br><br>
![CICD drawio](https://github.com/arasdk/jaffle-shop-ado-slim-ci/assets/145650154/7c44475a-bca4-443b-9716-5cff1ca88c8c)
<br><br>



## Pipeline Details

### Main Branch Pipeline

- **Trigger:** Scheduled (6:30 AM UTC, Monday to Friday).
- **Environment:** Runs on an Ubuntu VM.
- **Steps:**
  - **Install dbt and Python:** Uses the `template-steps-init-dbt.yml` template to set up the Python environment and install dbt along with the necessary dependencies.
  - **Build dbt Models:** Executes `dbt build` to compile and run all dbt models, tests, and documentation within the "ci_main" schema.
  - **Upload Manifest:** Uploads the dbt manifest file to a specified Azure storage account for use in the Pull Request Pipeline.

### Pull Request Pipeline

- **Trigger:** Runs on each PR.
- **Environment:** Runs on an Ubuntu VM.
- **Steps:**
  - **Install dbt and Python:** Similar to the Main Branch Pipeline, it sets up the environment for dbt.
  - **Download Manifest:** Downloads the previously generated dbt manifest from the main branch to use for deferring unchanged models.
  - **Build dbt Models (Deferred):** Executes `dbt build` with the `--defer` option, leveraging the "ci_main" schema to only rebuild and test models directly affected by the PR.

## Configuration

The pipelines utilize a `profiles.yml` file for dbt configuration, specifying connection details to the Databricks data warehouse through environment variables. This file is referenced by the Azure DevOps Pipelines via the `DBT_PROFILES_DIR` environment variable, which is used by the dbt CLI tool to locate the profile.

## Usage
The dbt profile used in this sample code is using Databricks SQL Warehouse as the Data Warehouse. This can easily be replaced with your data warehouse of choice by following the these steps: 

- Set up a storage account in Azure for manifest file management
- Configure an Azure DevOps Service Connection to enable the DevOps pipeline to access the storage account
- Add the two pipelines to the DevOps project
- Go to the Repository settings -> branch policies of your DevOps project and specify a build validation rule for the main branch in your project. Configure the `azure-pipelines-dbt-slim-ci.yml` pipeline to run as part of the build validation rule.
- Modify `profiles.yml` to use the appropriate settings for your data warehouse
- Modify `template-steps-init-dbt.yml` to intall your preferred dbt-adapter instead of `dbt-databricks`
- Set the correct environment variables for your dbt profile in `azure-pipelines-dbt-ci-main.yml` and `azure-pipelines-dbt-slim-ci.yml`

If you want to experiment with the pipelines, you can fork this repository into your own GitHub respository, and then add the two pipelines in Azure DevOps. The pipelines should work out of the box on the Jaffle Shop dbt project included here given the above modifications. For this to work you need to load the seed files from the `jaffle-data` folder manually to your data warehouse before running the pipelines.

To use these pipelines in your project, ensure that your Azure DevOps environment is configured with the necessary permissions and variables, including access to the Azure storage account for manifest file management. Modify the `profiles.yml` and pipeline YAML files as needed to match your dbt project and data warehouse setup.

**Note**: You need to setup an Azure DevOps Service Connection to enable the DevOps pipeline to access azure resources such as the storage account. The service connection needs to be granted `Storage Blob Data Contributor` on the storage account.
