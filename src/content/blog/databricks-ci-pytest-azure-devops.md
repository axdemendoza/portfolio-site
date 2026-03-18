---
title: "End-to-End CI for Databricks: Automated PyTest Pipelines with Azure DevOps"
slug: "databricks-ci-pytest-azure-devops"
pubDate: "2026-03-16"
description: "A step-by-step guide to running PyTest inside Databricks and publishing test results back to Azure DevOps."
---

Continuous integration (CI) is a foundational software engineering practice: developers merge code into a shared repository, and automated tests validate those changes before they move forward.

In traditional applications, that is usually straightforward. In Databricks, it often is not.

Your code may depend on:

- Spark sessions
- Databricks clusters
- Production-like data environments

Because of that, tests do not always run meaningfully in a standard CI runner. In many cases, they need to execute inside Databricks and report results back to your CI system.

In practice, this means running real tests inside Databricks directly from a pull request and automatically failing the pipeline if they break.

This guide walks through how to build a clean end-to-end workflow that:

- Runs `pytest` inside Databricks
- Triggers from Azure DevOps pipelines
- Publishes results back to Azure DevOps with JUnit XML

## What You Will Build

By the end of this setup, you will have a pipeline that:

1. Triggers from a branch or pull request in Azure DevOps
2. Starts a Databricks test run
3. Executes your `pytest` suite in the Databricks environment
4. Exports a JUnit XML report
5. Publishes test results directly in Azure DevOps

## What This Solves in Real-World Teams

In many data and ML teams, testing breaks down at the exact point where it matters most: once code depends on Spark, Databricks clusters, and production-like environments.

A local `pytest` run is useful, but it often is not enough to validate the actual execution path your code will take in production. Teams end up with a gap between development workflows and deployment reality.

This setup closes that gap by giving you a repeatable CI workflow that:

- runs tests in the environment your code actually depends on
- reports results back to the same Azure DevOps workflow engineers already use
- blocks pull requests when critical tests fail
- makes Databricks development feel much closer to standard software engineering

The result is a cleaner developer experience, earlier bug detection, and more confidence when shipping changes to shared data platforms.

## End-to-End Flow

![High-level CI flow for Databricks unit testing with Azure DevOps](/images/databricks-ci/ci-flow.png)

*High-level CI flow for validating Databricks code from an Azure DevOps pull request. The pipeline invokes a Databricks job, runs the PR source branch tests, exports a JUnit XML report, and publishes results back to Azure DevOps for pass/fail enforcement.*

---

## Step 0: Prerequisites

Before getting started, make sure you have:

- A Databricks workspace
- A project with `pytest`-based tests
- Access to Azure DevOps repositories and pipelines
- A Databricks cluster available for test execution

### Important: Spark and PyTest

`pytest` tests do not have access to a Spark session by default. If you want to validate Spark-based logic, define a shared fixture:

```python
import pytest
from pyspark.sql import SparkSession

@pytest.fixture(scope="session")
def spark():
    return SparkSession.builder.master("local[*]").getOrCreate()
```

Then use it in your tests:

```python
def test_example(spark):
    df = spark.createDataFrame([(1,)], ["value"])
    assert df.count() == 1
```

## Step 1: Create a Test Runner for CI

We need a way to trigger `pytest` inside Databricks and export results in a format Azure DevOps can consume.

### 1.1 Create a Test Runner Script

Create the following file:

```text
my_project/tests/run_tests.py
```

Add:

```python
import pytest

pytest.main([
    "--junit-xml",
    "/Workspace/Repos/my_project/tests/reports/report.xml",
    "."
])
```

This script:

- Runs your full test suite
- Writes results in JUnit XML format
- Saves the report to `tests/reports/report.xml`

That XML file is the key handoff between Databricks and Azure DevOps.

Azure DevOps natively understands JUnit format, which allows it to display structured test results and enforce pipeline failure conditions.

## Step 2: Configure the Databricks CLI

This step is optional, but useful for local validation and debugging before integrating with Azure DevOps.

Databricks supports multiple authentication methods (PAT, OAuth, service principals). For simplicity, this example uses a personal access token (PAT).

### 2.1 Generate a Personal Access Token

![Databricks PAT creation screen](/images/databricks-ci/databricks-pat.png)

Navigate to:

`Databricks -> User Settings -> Developer -> Access Tokens`

Create a new token and save the key.

### 2.2 Configure the CLI

Run:

```bash
databricks configure
```

Enter:

- Databricks host
- Personal access token

![Databricks config](/images/databricks-ci/db-configure.png)

## Step 3: Validate with a Databricks Job

Before integrating with Azure DevOps, it is helpful to validate that your tests run correctly inside Databricks in isolation.

In this step, we will trigger a Databricks job directly from the CLI using a local JSON definition. This allows you to debug test execution without involving the CI pipeline.

### 3.1 Create a Job Definition

Create a local file (for example `databricks-run.json`) with the following contents:

```json
{
  "name": "run_unit_tests",
  "tasks": [
    {
      "task_key": "run-test-notebook",
      "notebook_task": {
        "notebook_path": "Workspace/Repos/.../run_tests"
      },
      "existing_cluster_id": "<cluster-id>"
    }
  ]
}
```

This defines a Databricks job that runs your test notebook on an existing cluster.

### 3.2 Submit the Job from the CLI

Run the following command locally:

```bash
databricks jobs submit --json @databricks-run.json
```

This sends the job definition to Databricks, which then executes the notebook remotely on the specified cluster.


### 3.3 Validate the Output

After the job completes, go to Databricks and confirm that the test report was generated:

```text
/Workspace/repos/my_project/tests/reports/report.xml
```

If this file exists, it means:
- your tests successfully executed inside Databricks
- your JUnit XML report is being generated correctly

At this point, the Databricks execution layer is working as expected, and you are ready to integrate it into an Azure DevOps pipeline.

This step is optional but highly recommended. It isolates issues in your Databricks environment before introducing additional complexity from CI/CD orchestration.

## Step 4: Create the Azure DevOps Pipeline

Now we connect everything into an Azure DevOps pipeline that orchestrates the full test workflow:

- Determine which branch to test  
- Trigger a Databricks job to run tests  
- Export the test results  
- Fail the pipeline if tests do not pass  

Navigate to:

`Pipelines -> New Pipeline`

Then select:

- Azure Repos Git  
- Your repository  
- Python package  

---

### 4.1 Determine Which Branch to Test

When a pipeline runs, it needs to know which branch’s code to execute.

This is especially important for pull requests, where you want to test the **source branch being merged**, not the target branch.

```yaml
variables:
  - name: branch_name
    ${{ if startsWith(variables['Build.SourceBranch'], 'refs/heads/') }}:
      value: ${{ replace(variables['Build.SourceBranch'], 'refs/heads/', '') }}
    ${{ if startsWith(variables['Build.SourceBranch'], 'refs/pull/') }}:
      value: ${{ replace(variables['System.PullRequest.SourceBranch'], 'refs/heads/', '') }}
```

This logic ensures:

- manual runs use the current branch
- pull request runs use the PR source branch

### 4.2 Run Tests in Databricks

Next, we invoke Databricks directly from the pipeline using a Bash task.

```yaml
- task: Bash@3
  inputs:
    targetType: 'inline'
    script: |
      curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh

      databricks configure --host $(DATABRICKS_HOST) --token $(DATABRICKS_TOKEN)
      echo $(branch_name)
      databricks repos update /Workspace/repos/my_project --branch $(branch_name)
      databricks jobs submit --json '{
              "run_name": "run_unit_tests",
              "tasks": [
                {
                  "task_key": "run-test-notebook",
                  "notebook_task": {
                    "notebook_path": "Workspace/Repos/.../run_tests"
                  },
                  "existing_cluster_id": "<cluster-id>"
                }
              ]
            }'
```

This approach avoids rebuilding environments in the CI runner and instead executes tests directly in the Databricks workspace where the code runs in practice.

This step does several things:
- installs the Databricks CLI in the pipeline environment
- authenticates using secure pipeline variables
- checks out the correct branch
- triggers the Databricks job that runs your test notebook

At this point, your tests are executing remotely inside Databricks.

### 4.3 Export Test Results from Databricks

Once tests complete, we need to retrieve the JUnit XML report.

```yaml
databricks workspace export /tests/reports/report.xml report.xml
```

This command:
- pulls the XML report from Databricks
- saves it locally in the pipeline environment

This file is the bridge between Databricks execution and Azure DevOps test reporting.

### 4.4 Publish Results to Azure DevOps

```yaml
- task: PublishTestResults@2
  inputs:
    testResultsFiles: 'report.xml'
    testRunTitle: 'Databricks PyTest Results'
    failTaskOnFailedTests: true
```

Azure DevOps natively understands JUnit format, which allows it to:
- display structured test results
- mark the pipeline as failed if tests fail

This is what enables automated enforcement of test quality on pull requests. Using a standard format like JUnit avoids custom parsing and allows CI systems to handle reporting and failure conditions automatically.

### 4.5 Validate the Pipeline

Once everything is configured:
1. Run the pipeline manually to confirm it executes
2. Open a pull request and verify the pipeline triggers automatically
3. Intentionally break a test and confirm the pipeline fails
4. Fix the test and confirm the pipeline passes

At this point, you have a fully functioning CI pipeline for Databricks workflows.

---

## Full Pipeline YAML (Optional Reference)

For completeness, here is the full pipeline configuration assembled from the components above:

```yaml
variables:
  - name: branch_name
    ${{ if startsWith(variables['Build.SourceBranch'], 'refs/heads/') }}:
      value: ${{ replace(variables['Build.SourceBranch'], 'refs/heads/', '') }}
    ${{ if startsWith(variables['Build.SourceBranch'], 'refs/pull/') }}:
      value: ${{ replace(variables['System.PullRequest.SourceBranch'], 'refs/heads/', '') }}

# trigger when PR created on the main branch
trigger:
- main

# provision ubuntu VM
pool: 
  vmImage: 'ubuntu-latest'

steps:
# configure databricks CLI, checkout PR.source branch, and run notebook job
- task: Bash@3
  inputs:
    targetType: 'inline'
    script: |
      curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh

      databricks configure --host $(DATABRICKS_HOST) --token $(DATABRICKS_TOKEN)
      echo $(branch_name)
      databricks repos update /Workspace/repos/my_project --branch $(branch_name)
      databricks jobs submit --json '{
              "run_name": "run_unit_tests",
              "tasks": [
                {
                  "task_key": "run-test-notebook",
                  "notebook_task": {
                    "notebook_path": "Workspace/Repos/.../run_tests"
                  },
                  "existing_cluster_id": "<cluster-id>"
                }
              ]
            }'

      databricks workspace export /tests/reports/report.xml report.xml

- task: PublishTestResults@2
  inputs:
    testResultsFormat: 'JUnit'
    testResultsFiles: 'report.xml'
    testRunTitle: 'Databricks PyTest Results'
    failTaskOnFailedTests: true
```
## Step 5: Enforce CI on Pull Requests

Navigate to:

```text
Project Settings -> Repositories -> Policies -> Main Branch
```

Enable build validation and configure it as:
- Trigger: Automatic
- Policy Requirement: Required

![Azure DevOps build validation policy](/images/databricks-ci/build-policy.png)

*Configuring build validation to require passing tests before allowing merges.*

This ensures that:
- every pull request automatically runs the pipeline
- merges are blocked unless all tests pass

This is what turns your pipeline from a passive check into an enforced quality gate for every code change.

## Step 6: Validate the End-to-End Flow

Run through a quick sanity check:

1. Open a pull request and confirm the pipeline triggers automatically  
2. Intentionally break a test and confirm the pipeline fails  
3. Fix the test and confirm the pull request can merge  

At this point, your CI pipeline is actively enforcing test quality on every code change.

---

## Final Thoughts

This setup provides:

- Real CI coverage for Databricks workflows
- Better confidence in production code changes
- Automated quality enforcement for pull requests

More importantly, it addresses one of the most common pain points in data engineering: validating code that depends on distributed systems while still maintaining the rigor of modern CI practices.

Instead of treating Databricks as a special-case environment, this approach brings it into the same testing and deployment discipline expected in production software systems.
