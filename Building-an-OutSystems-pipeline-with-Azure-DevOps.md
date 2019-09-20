This guide provides a step-by-step description of how to implement a [recommended continuous delivery pipeline](https://success.outsystems.com/Documentation/Development_FAQs/How_to_build_an_OutSystems_continuous_delivery_pipeline) for OutSystems applications by leveraging the [LifeTime Deployment API](https://success.outsystems.com/Documentation/11/Reference/OutSystems_APIs/LifeTime_Deployment_API_v2) along with [Azure DevOps](https://azure.microsoft.com/en-us/services/devops/).

## Azure pipelines release flow

The role of a "deployment pipeline" is to ensure that code and infrastructure are always in a deployable state, and that all code checked in to trunk can be safely deployed into production.

The goal of the **Build Pipeline** is to provide everyone in the value stream, especially developers, the fastest possible feedback if a release candidate has taken them out of a deployable state. This could be a change to the code, to any of the environments, to the automated tests, or even to the pipeline infrastructure.

The Build Pipeline also stores information about which tests were performed on which build and what the test results were. In combination with the information from the version control history, we can quickly determine what caused the deployment pipeline to break and, likely, how to fix the error.

On the other hand, the **Release Pipeline** serves as the "push-button" device mentioned before, where all the necessary actions for successfully deploying to production a release candidate that has gone through the Build Pipeline are carried out.

> **Note**
>
> Most of the workload throughout the pipeline is performed by calling a set of functions from the [outsystems-pipeline](https://pypi.org/project/outsystems-pipeline/) Python package, distributed by OutSystems on the Python Package Index (PyPI) repository.

### Build Pipeline anatomy

The Build Pipeline is comprised of **4 sequential stages**, each one performing one or more actions as described in the following table:

| Stage | Actions performed |
|-------|-------------------|
| Install Dependencies | Install required Python dependencies for running pipeline activities. |
| Deploy to REG | Fetch the latest versions (tags) in DEV for the configured Applications and deploy them to REG environment. |
| Run Regression | Generate Python script for running BDD test scenarios using unittest module.<br/><br/>Run unit test suite and publish test results report. |
| Deploy to ACC | Deploy the latest versions (tags) for the configured Applications (excluding test apps) from REG to ACC environment. |

By leveraging the test-execution REST API, the pipeline is able to run tests written with the BDD Framework in the list of configured test applications. The outcome of this action is presented as a JUnit test report that seamlessly integrates with the Azure DevOps UI, as shown in the following picture:

![Test report](images/azure-test-report-success.png)

> **Note**
>
> To learn more about running tests written with the BDD Framework using the test-execution REST API, please check [this article](https://www.outsystems.com/blog/posts/automate-bddframework-testing/).

Whenever **Run Regression** fails, the ongoing pipeline run is marked as failed and all subsequents stages are skipped, thus preventing a release candidate version to proceed further down the pipeline. The pipeline test report will also display which tests failed and why, for easier troubleshooting of the regression error. 

![Test report with failed regression test](images/azure-test-report-fail.png)

### Release Pipeline anatomy

The release pipeline is comprised of **3 sequential stages**, each one performing one or more actions as described in the following table:

| Stage | Actions performed |
|-------|-------------------|
| Deploy to ACC | Deploy the latest versions (tags) for the configured Applications (excluding test apps) from REG to ACC environment.<br/><br/>Wait for input from an authorized user to accept changes and proceed until production. |
| Deploy Dry-Run | Deploy the latest versions (tags) for the configured Applications from ACC to PRE environment. |
| Deploy Production | If the dry-run is successful, immediately trigger the deployment to PRD of the latest versions (tags) for the configured Applications. |

Whenever the pipeline reaches **Accept Changes** after the successful deploy on ACC, Azure Pipelines halts the pipeline execution until an authorized user makes the decision to accept (or reject) the release candidate version and proceed until production without further human intervention.

![Azure DevOps UI waiting for input of an authorized user](images/azure-approval-input.png)

## Prerequisites

Please confirm that you have the following prerequisites before following the steps described in this guide:

* Make sure you have the [necessary rights to configure pipelines](https://docs.microsoft.com/en-us/azure/devops/pipelines/policies/set-permissions?view=azure-devops) in your Azure DevOps project.

* Have at least one [Azure Pipelines self-hosted agent](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/agents?view=azure-devops#install) with:
  * Python 3 installed

    Make sure that you also install the `pip` package installer.<br/>
    On Windows systems, also make sure to activate the option "Add Python to PATH" in the installation wizard.

  * Access to PyPI.org

  * HTTPS connectivity with LifeTime

  * HTTPS connectivity with the front-end servers that are going to run test suites

## Step-by-step configuration guide

### 1. Publish CI/CD probe in Regression environment

To retrieve environment-specific information that is required when running the continuous delivery pipeline, the [CI/CD Probe](https://www.outsystems.com/forge/component-overview/6528/ci-cd-probe) Forge component must be installed on the Regression environment of your deployment pipeline.

To install the CI/CD probe, download the [CI/CD Probe matching your Platform Server version](https://www.outsystems.com/forge/component-versions/6528) and publish it on the Regression environment using Service Center. Alternatively, you can install the component directly from the Service Studio interface.

![CI/CD Probe on OutSystems Forge](images/forge-cicd-probe.png)

> **Note**
>
> For the time being, the CI/CD Probe is only used for discovering the endpoints of existing BDD test scenarios in the target environment. Additional functionality may be added to the CI/CD Probe in future iterations.

### 2. Set up variables

In Azure, **variable groups** are a convenient way to to make sets of reusable variables available across multiple pipelines.

In this guide we'll create and use a single variable group that will provide all infrastructure context to run the pipelines.

#### 2.1. Create a variable group

From the Azure Pipelines Dashboard, open the "Library" tab to see a list of existing variable groups for your project and choose "+ Variable group".

Enter a name and description for the variable group and make it accessible to any pipeline by setting the option "Allow access to all pipelines".

#### 2.2. Register LifeTime authentication token as secret variable

You need to [create a LifeTime service account and generate an authentication token](https://success.outsystems.com/Documentation/11/Reference/OutSystems_APIs/LifeTime_Deployment_API_v2/REST_API_Authentication) to authenticate all requests to the Deployment API.

From the variable group previously created, select "+ Add" and provide the following configuration values:

* **Name:** LifeTimeServiceAccountToken
* **Value:** &lt;your LifeTime authentication token&gt;

Click on the "lock" icon at the end of the row to encrypt and securely store the value. (The values of secret variables are stored securely on the server and cannot be viewed by users after they are saved. After being encrypted the values can only be replaced.)

![Add Credentials](images/azure-add-credentials.png)

#### 2.3. Register environment variables

For a smooth run of the pipeline you'll need to provide some context about your OutSystems factory and global values to be used by the pipelines.

From the variable group previously created, select "+ Add" and provide the following configuration values:

| Name | Value | Example value |
|------|-------|---------------|
| LifeTimeHostname | &lt;Hostname of LifeTime environment&gt; | lifetime.example.com |
| LifeTimeAPIVersion | &lt;Version of LifeTime Deployment API to use&gt; | 2 |
| DevelopmentEnvironment | &lt;Name of Development environment&gt; | Development |
| RegressionEnvironment | &lt;Name of Regression environment&gt; | Regression |
| AcceptanceEnvironment | &lt;Name of Acceptance environment&gt; | Acceptance |
| PreProductionEnvironment | &lt;Name of Pre-Production environment&gt; | Pre-Production |
| ProductionEnvironment | &lt;Name of Production environment&gt; | Production |
| ProbeEnvironmentURL | &lt;URL of environment where CI/CD probe is deployed&gt; | https://regression-env.example.com/ |
| BddEnvironmentURL | &lt;URL of environment where BDD tests will run automatically&gt; | https://regression-env.example.com/ |
| ArtifactsBuildFolder | &lt;Folder that will store all outputs from the Build pipeline to be used by the Release pipeline&gt; | Artifacts |
| ArtifactName | &lt;Name specification for the compressed outputs to be used by the Release pipeline&gt; | manifest |
| ArtifactsReleaseFolder | &lt;Folder that will store the outputs from the Build pipeline&gt; | $(System.DefaultWorkingDirectory) |
| OSPipelineVersion | &lt;Outsystems Python package version&gt; | 0.2.18 |

### 3. Create Build and Release Pipelines

To orchestrate the flow of activities for the continuous delivery pipeline, as described in the introduction, you'll need to create two pipelines: Build and Release Pipelines.

#### 3.1. Create a Build Pipeline using template definition file

The easiest way to do this is by providing a YAML file containing the build pipeline definition. A template YAML file for the OutSystems continuous delivery pipeline [is provided here](https://github.com/OutSystems/outsystems-pipeline/tree/master/examples/azure_devops).

It is highly advisable to store your template YAML file using a version control system such as Git, as this ensures that you are able to keep track of any changes made to your pipeline definition going forward. Additionally, any other supporting artifacts that you may need to run your continuous delivery pipeline can also be stored in a single location alongside the YAML file, and synced to the pipeline workspace folder on every run.

##### 3.1.1. Create Build Pipeline

From the Azure Pipeline Dashboard, navigate to the **Pipelines** tab, under **Builds** page, select **+ New**, **New build pipeline**, choose "Use the classic editor to create a pipeline without YAML", then select the version control system (Source) where you've previously stored the YAML file.

> **Note**
>
> We have chosen the classic editor to have a streamlined configuration. It's also possible to configure by selecting the "Existing Azure Pipelines YAML file" option but you will have to cancel the pipeline execution immediately after you save it since it will attempt to run it immediately.

Once you have configured the version control system, choose YAML for **Configuration as code**.

![Choose YAML for Configuration as code](images/azure-yaml-configuration-as-code.png)

From the Pipeline tab define the name as "&lt;YourProduct&gt;-Build-Pipeline", choose the **Default Agent Pool** and select the previously stored YAML file.

![Configure Build pipeline](images/azure-configure-build-pipeline.png)

##### 3.1.2. Configure Build Pipeline variables

From the **Variables** tab, navigate to the variable groups, select the **Link variable group** button and choose the previously created variable group.

![Link variable group](images/azure-link-variable-group.png)

From the **Variables** tab, navigate to the **Pipeline Variables**,  select **+ Add** and provide the following configuration values:

| Name | Value | Settable at queue time |
|------|-------|------------------------|
| ApplicationScope | Leave blank, the value will be fetched from LifeTime Trigger Plugin | Yes |
| ApplicationScopeWithTests | Leave blank, the value will be fetched from LifeTime Trigger Plugin | Yes |
| TriggeredBy | Leave blank, the value will be fetched from LifeTime Trigger Plugin | Yes |

![Configure Build Pipeline variables](images/azure-build-pipeline-variables.png)

From the **Save & Queue** tab choose **Save**.

![Save Build Pipeline variables](images/azure-build-pipeline-save-variables.png)

#### 3.2. Create a task group

A template JSON file for the task group [is provided here](https://github.com/OutSystems/outsystems-pipeline/tree/master/examples/azure_devops).

From the Azure Pipeline Dashboard, navigate to the **Pipelines** tab, under **Task groups** page, select **Import**, and upload the JSON template file. Then, save the newly created task group.

![Import task group](images/azure-import-task-group.png)

#### 3.3 Create a Release Pipeline

As Azure does not yet support pipeline as code for Release Pipelines, we have to configure it via the GUI.

From the Azure Pipeline Dashboard, navigate to the **Pipelines** tab, under **Releases** page, select **+ New**, **New release pipeline**, and choose **Empty job**.

##### 3.3.1. Get artifacts from Build Pipeline

