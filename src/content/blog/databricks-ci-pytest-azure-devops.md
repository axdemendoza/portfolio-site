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
    "--junit-xml=tests/reports/report.xml",
    "tests"
])
```

This script:

- Runs your full test suite
- Writes results in JUnit XML format
- Saves the report to `tests/reports/report.xml`

That XML file is the key handoff between Databricks and Azure DevOps. Azure DevOps natively understands JUnit format, which allows it to display structured test results and enforce pipeline failure conditions.

## Step 2: Configure the Databricks CLI

This step is optional for manual experimentation, but strongly recommended. It makes local validation easier and is useful when you automate the workflow later.

### 2.1 Generate a Personal Access Token

![Databricks PAT creation screen](/images/databricks-ci/databricks-pat.png)

Navigate to:

`Databricks -> User Settings -> Developer -> Access Tokens`

Create a new token and copy the value.

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

Before integrating with Azure DevOps, confirm that your tests run correctly inside Databricks.

### 3.1 Create a Job Definition

```json
{
  "name": "run_unit_tests",
  "tasks": [
    {
      "task_key": "run-test-notebook",
      "notebook_task": {
        "notebook_path": "/Repos/.../run_tests"
      },
      "existing_cluster_id": "<cluster-id>"
    }
  ]
}
```

### 3.2 Run the Job

```bash
databricks jobs create --json-file job.json
databricks jobs run-now --job-id <id>
```

### 3.3 Validate the Output

Confirm that this file exists:

```text
tests/reports/report.xml
```

If this file is generated, your Databricks execution layer is working correctly.

## Step 4: Create the Azure DevOps Pipeline

Now we connect everything into an Azure DevOps pipeline that will:
- trigger the Databricks job
- retrieve test results
- enforce pass/fail behavior on pull requests

Navigate to pipelines, then select:

- Azure Repos Git
- Your repository
- Python package

### 4.1 Handle Branch Logic

We need to ensure the pipeline tests the correct branch, especially for pull requests.

```yaml
variables:
  - name: branch_name
    ${{ if startsWith(variables['Build.SourceBranch'], 'refs/heads/') }}:
      value: ${{ replace(variables['Build.SourceBranch'], 'refs/heads/', '') }}
    ${{ if startsWith(variables['Build.SourceBranch'], 'refs/pull/') }}:
      value: ${{ replace(variables['System.PullRequest.SourceBranch'], 'refs/heads/', '') }}
```

Why this matters:

- It handles both manual runs and pull request validation
- It ensures the pipeline checks the actual feature branch

### 4.2 Trigger the Databricks Job from the Pipeline

```yaml
- task: Bash@3
  inputs:
    targetType: 'inline'
    script: |
      pip install databricks-cli

      databricks configure --token <<EOF
      $(DATABRICKS_HOST)
      $(DATABRICKS_TOKEN)
      EOF

      echo "Branch: $(branch_name)"
      git checkout $(branch_name)

      databricks jobs run-now --job-id <job-id>
```

### 4.3 Export the Test Results

```bash
databricks workspace export /tests/reports/report.xml report.xml
```

### 4.4 Publish the Test Results

```yaml
- task: PublishTestResults@2
  inputs:
    testResultsFiles: 'report.xml'
    testRunTitle: 'Databricks PyTest Results'
    failTaskOnFailedTests: true
```

## Step 5: Enforce CI on Pull Requests

Navigate to:

```text
Project Settings -> Repositories -> Policies -> Main Branch
```

Enable:

- Build validation
- Automatic trigger
- Required

## Step 6: Validate the End-to-End Flow

Run through a quick sanity check:

1. Open a pull request and confirm the pipeline triggers
2. Intentionally break a test and confirm the pipeline fails
3. Fix the test and confirm the pull request can merge

## Final Thoughts

This setup gives you:

- Real CI coverage for Databricks workflows
- Better confidence in production code changes
- Automated quality enforcement for pull requests

More importantly, it addresses one of the most common pain points in data engineering: validating code that depends on distributed systems while still maintaining the rigor of modern CI practices.

Instead of treating Databricks as a special-case environment, this approach brings it into the same testing and deployment discipline expected in production software systems.
