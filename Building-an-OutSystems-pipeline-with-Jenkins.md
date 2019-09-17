This guide provides a step-by-step description of how to implement a [recommended continuous delivery pipeline](faq.md) for OutSystems applications by leveraging the [LifeTime Deployment API](https://success.outsystems.com/Documentation/11/Reference/OutSystems_APIs/LifeTime_Deployment_API_v2) along with Jenkins, an open-source automation server.

## Prerequisites

Confirm that you have the following prerequisites installed before executing the steps described in this guide:

| Prerequisite | Version used for this guide | Notes |
|--------------|-----------------------------|-------|
| Jenkins Automation Server | 2.176.1 | [Install Jenkins](https://jenkins.io/doc/book/installing/) on a separate machine from the OutSystems platform. |
| [Blue Ocean](https://plugins.jenkins.io/blueocean) (plugin) | 1.17.0 | |
| [JUnit](https://plugins.jenkins.io/junit) (plugin) | 1.28 | |
| [Pyenv Pipeline](https://plugins.jenkins.io/pyenv-pipeline) (plugin) | 2.1.1 | |
| Python 3 (installed on the agent) | 3.7.3 | Make sure that you also install the `pip` package installer.<br/><br/>On Windows systems, also make sure to activate the option **Add Python to PATH** in the installation wizard. |
| Git (installed on the agent) | 2.22.0 | Required when storing the pipeline definition script in a Git repository. |

Refer to the [Jenkins documentation](https://jenkins.io/doc/book/managing/plugins/) on how to install Jenkins plugins.

## Step-by-step configuration guide

### Register LifeTime Authentication Token as Jenkins Credential

You need to [create a LifeTime Service Account and generate an Authentication Token](https://success.outsystems.com/Documentation/11/Reference/OutSystems_APIs/LifeTime_Deployment_API_v2/REST_API_Authentication) to authenticate all requests to the Deployment API.

This token can then be configured as a Jenkins credential, allowing it to be reused throughout your pipelines via the corresponding credential ID, while keeping it secure and accessible only by authorized users.

From the Jenkins Dashboard, go to **Credentials**, select **System** and **Global credentials (unrestricted)** (or you can opt to store the credential in a different domain).

Select **Add Credentials** and provide the following configuration values:

* **Kind:** _Secret text_
* **Scope:** _Global_
* **Secret:** _&lt;your LifeTime authentication token&gt;_
* **ID:** _LifeTimeServiceAccountToken_
* **Description:** _Authentication token required for invoking LifeTime Deployment API._

![Add Credentials](images/jenkins-add-credentials.png?width=800)

### Publish CI/CD Probe on Regression environment

To retrieve environment-specific information that is required when running the continuous delivery pipeline, the [CI/CD Probe](https://www.outsystems.com/forge/component-overview/6528/ci-cd-probe) Forge component must be installed on the Regression environment of your deployment pipeline.

To install the CI/CD probe, download the [CI/CD Probe matching your Platform Server version](https://www.outsystems.com/forge/component-versions/6528) and publish it on the Regression environment using Service Center. Alternatively, you can install the component directly from the Service Studio interface.

![CI/CD Probe on OutSystems Forge](images/forge-cicd-probe.png?width=800)

> **Note**
>
> For the time being, the CI/CD Probe is only used for discovering the endpoints of existing BDD test scenarios in the target environment. Additional functionality may be added to the CI/CD Probe in future iterations.

### Create Jenkins Pipeline using template definition file

To orchestrate the flow of activities for the continuous delivery pipeline you’ll need to create a Jenkins pipeline. The easiest way to do this is by providing a [Jenkinsfile](https://jenkins.io/doc/book/pipeline/jenkinsfile/) text file containing the pipeline definition.

A template Jenkinsfile for the OutSystems continuous delivery pipeline [is provided here](https://github.com/OutSystems/outsystems-pipeline/tree/master/examples/jenkins).

Create your own copy of the supplied Jenkinsfile template and make sure that the following pipeline **environment variables** are properly set inside the newly created Jenkinsfile:

```
LifeTimeHostname=<Hostname of LifeTime environment>
LifeTimeAPIVersion=<Version of LifeTime Deployment API to use>
DevelopmentEnvironment=<Name of Development environment>
RegressionEnvironment=<Name of Regression environment>
AcceptanceEnvironment=<Name of Acceptance environment>
PreProductionEnvironment=<Name of Pre-Production environment>
ProductionEnvironment=<Name of Production environment>
AuthorizationToken=credentials('<ID of Jenkins credential that stores the LifeTime authentication token>')
ProbeEnvironmentURL=<URL of environment where CI/CD probe is deployed>
BddEnvironmentURL=<URL of environment where BDD tests will run automatically>
```

It is highly advisable to store your custom Jenkinsfile using a version control system such as Git, as this ensures that you are able to keep track of any changes made to your pipeline definition going forward. Additionally, any other supporting artifacts that you may need to run your continuous delivery pipeline can also be stored in a single location alongside the Jenkinsfile, and synced to the pipeline workspace folder on every run.

From the Jenkins Dashboard, go to **New Item**, select **Pipeline** and name it _&lt;YourApp&gt;-CD-Pipeline_.

![Jenkins Pipeline](images/jenkins-pipeline.png?width=800)

> **Note**
>
> To better organize your Jenkins dashboard you can opt to create the Jenkins pipeline inside an existing **Folder**.

In the **Pipeline** section, select option **Pipeline script from SCM** in the **Definition** field and provide the following configuration values:

* **SCM:** _&lt;Source control management system that hosts the Jenkinsfile pipeline definition&gt;_
* **Repository URL:** _&lt;URL of the Jenkinsfile in the source control management system&gt;_
* **Credentials:** _&lt;Credentials used by Jenkins to fetch content from the source control management system&gt;_

The Jenkins pipeline created using the provided pipeline definition file contains 6 sequential stages, each one performing one or more actions as described in the following table:

| Stage | Actions performed |
|-------|-------------------|
| **Install Python Dependencies** | Install required Python dependencies for running pipeline activities. |
| **Get and Deploy Latest Tags** | Fetch the latest versions (tags) in DEV for the configured Applications and deploy them to REG environment. |
| **Run Regression** | Generate Python script for running BDD test scenarios using unittest module.<br/><br/>Run unit test suite and publish test results report. |
| **Accept Changes** | Deploy the latest versions (tags) for the configured applications (excluding test apps) from REG to ACC environment.<br/><br/>Wait for input from an authorized user to accept changes and proceed until production. |
| **Deploy Dry-Run** | Deploy the latest versions (tags) for the configured applications from ACC to PRE environment. |
| **Deploy Production** | If the dry-run is successful, immediately trigger the deployment to PRD of the latest versions (tags) for the configured applications. |

> **Note**
>
> Most of the workload throughout the pipeline is performed by calling a set of functions from the [outsystems-pipeline](https://pypi.org/project/outsystems-pipeline/) Python package, distributed by OutSystems on the Python Package Index (PyPI) repository.

The following picture shows a successful pipeline run using the Blue Ocean plugin:

![Successful pipeline run](images/jenkins-blue-ocean-success.png?width=800)

By leveraging the test-execution REST API, the Jenkins pipeline is able to [run tests written with the BDD Framework](https://www.outsystems.com/blog/posts/automate-bddframework-testing/) in the list of configured test applications. The outcome of this action is presented as a JUnit test report that seamlessly integrates with the Jenkins UI, as shown in the following picture:

![JUnit test report](images/jenkins-junit-report-success.png?width=800)

Whenever the **Run Regression** fails, the ongoing pipeline run is marked as unstable and all subsequents stages are skipped, thus preventing a release candidate version to proceed further down the pipeline. The pipeline test report displays which tests failed and why, for easier troubleshooting of the regression errors.

![Pipeline run with failed regression](images/jenkins-blue-ocean-fail.png?width=800)

![JUnit test report with failed regression test](images/jenkins-junit-report-fail.png?width=800)

Whenever the pipeline reaches the **Accept Changes** stage, Jenkins [halts the pipeline execution](https://jenkins.io/doc/pipeline/steps/pipeline-input-step/) until an authorized user makes the decision to either accept or reject the release candidate version and proceed until production without further human intervention.

![Pipeline run waiting for the decision of an authorized user](images/jenkins-blue-ocean-halt.png?width=800)

![Jenkins UI waiting for input of an authorized user](images/jenkins-input.png?width=800)

This step in the pipeline definition serves as the "push-button" device mentioned in the Introduction section, where all the necessary actions for successfully deploying to production a release candidate that has gone through the deployment pipeline are carried out by pushing a button.

### Trigger pipeline execution remotely

Triggering subsequent pipeline runs can be made directly from the Jenkins UI, as needed. This approach, however, is not desirable as it would require provisioning a Jenkins user account for each person that is allowed to trigger the pipeline - in the worst case scenario, a Jenkins account per developer.

On the other hand, the purpose of the deployment pipeline is to reduce the amount of manual work required throughout the pipeline’s flow of activities.

To address this issue, the [Trigger Pipeline](https://www.outsystems.com/forge/component-overview/5670/trigger-pipeline) LifeTime plugin available on the OutSystems Forge automatically detects when new versions (tags) are available for a configured subset of LifeTime applications and triggers an associated Jenkins pipeline by leveraging the Jenkins REST API.

To install the Trigger Pipeline plugin, download the [Trigger Pipeline plugin matching your Platform Server version](https://www.outsystems.com/forge/component-versions/5670) and publish it to your LifeTime environment using Service Center. Alternatively, you can install the component directly from the Service Studio interface.

![Trigger Pipeline on OutSystems Forge](images/forge-trigger-pipeline.png?width=800)

After the plugin is successfully published in the LifeTime environment, select **Configure Triggers** from the plugin landing page in LifeTime and configure the following parameters:

* **Source Environment:** _&lt;Select your OutSystems Development environment&gt;_
* **Pipeline Server Type:** _Jenkins_
* **Pipeline Server Address:** _&lt;Your Jenkins instance base URL&gt;_
* **Pipeline Server Credentials:** _&lt;Credentials of a Jenkins user account with enough permissions for running pipeline jobs&gt;_

![Trigger Pipeline configuration](images/trigger-pipeline-config-jenkins.png?width=800)

One or more pipeline triggers can be configured by providing the following data:

* **Pipeline:** _&lt;Unique name that identifies the pipeline in Jenkins (including folders)&gt;_
* **Applications:** _&lt;List of LifeTime applications that will trigger the CI/CD pipeline, identifying which ones are Test applications&gt;_

![Configure applications that trigger the CI/CD pipeline](images/trigger-pipeline-details.png?width=800)

After the Trigger Pipeline plugin is properly configured, the dashboard screen will show the list of pipeline triggers, along with the current versions in Development of the LifeTime applications defined for each pipeline scope.  

Once there are new application versions available for triggering a pipeline, a button is shown that allows running the pipeline on-demand without the need to log in to Jenkins.

![List of pipelines that can be triggered on-demand](images/trigger-pipeline.png?width=800)

Alternatively, pipelines can be triggered automatically through the **CheckNewVersionsForPipeline** timer that periodically checks if there are new application versions in Development within the scope of each configured pipeline.

To enable this timer, go to the Service Center console of your LifeTime environment and configure a desirable schedule. The minimum configurable interval is 5 minutes.

![Configure timer CheckNewVersionsForPipeline](images/trigger-pipeline-timer.png?width=800)

> **Note**
>
> A pipeline cannot be triggered while there are still **pending** changes (within the pipeline scope) that have not yet been tagged in LifeTime. The reason for this is to avoid running the pipeline while the changeset is still open and the commit stage has not yet been finalized.
