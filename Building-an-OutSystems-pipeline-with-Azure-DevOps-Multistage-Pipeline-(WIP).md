This guide provides a step-by-step description of how to implement a [recommended continuous delivery pipeline](https://success.outsystems.com/Documentation/How-to_Guides/DevOps/How_to_build_an_OutSystems_continuous_delivery_pipeline) for OutSystems applications by leveraging the [LifeTime Deployment API](https://success.outsystems.com/Documentation/11/Reference/OutSystems_APIs/LifeTime_Deployment_API_v2) along with [Azure DevOps](https://azure.microsoft.com/en-us/services/devops/).

## Prerequisites

Please confirm that you have the following prerequisites before following the steps described in this guide:

* Make sure you have the [necessary rights to configure pipelines](https://docs.microsoft.com/en-us/azure/devops/pipelines/policies/set-permissions?view=azure-devops) in your Azure DevOps project.

* At least one [Azure Pipelines agent](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/agents?view=azure-devops) with:
  * Python 3 installed (preferably Python 3.7)

  * Access to PyPI.org

  * HTTPS connectivity with LifeTime

  * HTTPS connectivity with the front-end servers that are going to run test suites

> **Note:**
>
> If you are using self-hosted Windows OS agents, make sure that you also install the `pip` package installer.<br/>
> Also make sure to activate the option "Add Python to PATH" in the installation wizard.

## Step-by-step configuration guide

### 1. Set up variables

In Azure, **variable groups** are a convenient way to make sets of reusable variables available across multiple pipelines.<br/>
In this guide, we'll create and use a single variable group that will provide all infrastructure context to run the pipelines.

#### 1.1. Create a variable group

From the Azure Pipelines Dashboard, open the "Library" tab to see a list of existing variable groups for your project and choose "+ Variable group".<br/>
Enter a name and description for the variable group and make it accessible to any pipeline by setting the option "Allow access to all pipelines".

#### 1.2. Register LifeTime authentication token as a secret variable

You need to [create a LifeTime service account and generate an authentication token](https://success.outsystems.com/Documentation/11/Reference/OutSystems_APIs/LifeTime_Deployment_API_v2/REST_API_Authentication) to authenticate all requests to the LifeTimeAPI.

From the variable group previously created, select "+ Add" and provide the following configuration values:

* **Name:** LifeTime.ServiceAccountToken
* **Value:** &lt;your LifeTime authentication token&gt;

Click on the "lock" icon at the end of the row to encrypt and securely store the value. (The values of secret variables are stored securely on the server and cannot be viewed by users after they are saved. After being encrypted the values can only be replaced.)

![Add Credentials](images/azure-add-credentials.png)


#### 1.3. Register environment variables

For a smooth run of the pipeline, you'll need to provide some context about your OutSystems factory global values to be used by the pipelines.

From the variable group previously created, select "+ Add" and provide the following configuration values:

| Name | Value | Example value |
|------|-------|---------------|
| Agent.VMImage | &lt;Microsoft-hosted agent image&gt; | ubuntu-latest |
| LifeTime.Hostname | &lt;Hostname of LifeTime environment&gt; | lifetime.example.com |
| LifeTime.APIVersion | &lt;Version of LifeTime Deployment API to use&gt; | 2 |
| Environment.Development.Label | &lt;Name of Development environment as defined on LifeTime&gt; | Development |
| Environment.Regression.Label | &lt;Name of Regression environment as defined on LifeTime&gt; | Regression |
| Environment.Acceptance.Label | &lt;Name of Acceptance environment as defined on LifeTime&gt; | Acceptance |
| Environment.PreProduction.Label | &lt;Name of Pre-Production environment as defined on LifeTime&gt; | Pre-Production |
| Environment.Production.Label | &lt;Name of Production environment as defined on LifeTime&gt; | Production |
| Environment.Regression.Key| &lt;Id of the Azure DevOps Environment for Regression&gt; | reg-env |
| Environment.Acceptance.Key| &lt;Id of the Azure DevOps Environment for Acceptance&gt; | acc-env |
| Environment.PreProduction.Key| &lt;Id of the Azure DevOps Environment for Pre-Production&gt; | pre-env |
| Environment.Production.Key| &lt;Id of the Azure DevOps Environment for Production&gt; | prd-env |
| Artifacts.Folder | &lt;Folder that will store all pipeline outputs&gt; | Artifacts |
| Manifest.File | &lt;Name specification for the compressed pipeline manifest output&gt; | trigger_manifest.json |
| Manifest.Folder | &lt;Folder that will store the manifest outputs&gt; | trigger_manifest |
| OSPackage.Version | &lt;Outsystems Python package version&gt; | 0.5.0 |
| Python.Version | &lt;Python version to run scripts&gt; | 3.7 |
<br/>
You may additionally add the following configuration values depending on your integrations; choose the ones that apply to you:<br/>
<br/>

| Name | Value | Example value |
|------|-------|---------------|
| CICDProbe.EnvironmentURL | &lt;URL of environment where CI/CD probe is deployed&gt; | https://regression-env.example.com/ |
| BDDFramework.EnvironmentURL | &lt;URL of environment where BDD tests will run automatically&gt; | https://regression-env.example.com/ |
| ArchDashboard.ActivationCode | &lt;OutSystems License Activation Code&gt; | AAA.AAA.AAA.AAA.AAA.AAA.AAA.AAA |
| ArchDashboard.APIKey | &lt;API key to authenticate request&gt; |  |
| ArchDashboard.Hostname | &lt;Hostname of Architecture Dashboard&gt; | architecture.outsystems.com |
| ArchDashboard.Folder | &lt;Name specification for the compressed Architecture Dashboard outputs&gt; | techdebt_data |
| ArchDashboard.Thresholds.SecurityFindingsCount | &lt;Microsoft-hosted agent image&gt; | 10 |
| ArchDashboard.Thresholds.TechDebtLevel | &lt;Hostname of LifeTime environment&gt; | Medium |

### 2. Create a Multistage Pipeline

The easiest way to do this is by providing a YAML file containing the multistage pipeline definition. A template YAML file for the OutSystems continuous delivery pipeline is provided here (To be added later).

It is highly advisable to store your template YAML file using a version control system such as Git, as this ensures that you are able to keep track of any changes made to your pipeline definition going forward. Additionally, any other supporting artifacts that you may need to run your pipeline can also be stored in a single location alongside the YAML file, and synced to the pipeline workspace folder on every run.

> **Note:**
>
> You may want to edit the YAML template to match the name of the previously created Variable Group.<br/>
> If you're using self-hosted agents you may also want to edit the YAML template to define the pool name.

From the Azure Pipeline Dashboard, navigate to the **Pipelines** tab, under **Pipelines** page, click **New pipeline**, and select the version control system (Source) where you've previously stored the YAML file.

Once you have configured the version control system, choose **Existing Azure Pipelines YAML file**, select the template YAML file, and click Continue.

Review Pipeline YAML and click **Save** from the dropdown.

Click the **Edit** button, select **Triggers** from the More actions submenu and rename the pipeline to "&lt;YourProduct&gt;-Pipeline".

### 3. Publish OutSystems Forge Components

#### 3.1 Publish Properties Services in all environments (Optional)

Properties Services allows you to get and set Site Properties, Timers, REST References, and SOAP Reverences from the pipelines.<br/>
To install the Properties Services, download the [Properties Services](https://www.outsystems.com/forge/component-overview/3966/properties-services) and publish it on every environment that you'd like to have this functionality using Service Center. Alternatively, you can install the component directly from the Service Studio interface.<br/>


#### 3.2 Publish Properties Management in the LifeTime environment (Optional)

Properties Management provides the APIs to externally use the [Properties Services](https://www.outsystems.com/forge/component-overview/3966/properties-services) component.<br/>
To install the Properties Management, download the [Properties Management](https://www.outsystems.com/forge/component-overview/947/properties-management) and publish it on LifeTime environment using Service Center. Alternatively, you can install the component directly from the Service Studio interface.<br/>

#### 3.3 Publish CI/CD probe in the Regression environment

To retrieve environment-specific information that is required when running the continuous delivery pipeline, the [CI/CD Probe](https://www.outsystems.com/forge/component-overview/6528/ci-cd-probe) Forge component must be installed on the Regression environment of your deployment pipeline.

To install the CI/CD probe, download the [CI/CD Probe matching your Platform Server version](https://www.outsystems.com/forge/component-versions/6528) and publish it on the Regression environment using Service Center. Alternatively, you can install the component directly from the Service Studio interface.

### 4. Trigger pipeline execution remotely

The [Trigger Pipeline](https://www.outsystems.com/forge/component-overview/5670/trigger-pipeline) LifeTime plugin available on the OutSystems Forge automatically detects when new versions (tags) are available for a configured subset of LifeTime applications and triggers an associated Azure DevOps pipeline by leveraging the Azure DevOps REST API.

#### 4.1 Install the Trigger Pipeline LifeTime plugin

To install the Trigger Pipeline LifeTime plugin, download the [Trigger Pipeline version 2.4.0](https://www.outsystems.com/forge/component-versions/5670) and publish it to your LifeTime environment using Service Center. Alternatively, you can install the component directly from the Service Studio interface.

#### 4.2 Configure the Trigger Pipeline LifeTime plugin

Configure the Trigger plugin module in your Lifetime Service Center using the following steps.

#### 4.2.1 Integrations configuration

Access the integrations tab and configure the following REST APIs:

* **LifeTimeAPI:** _`https://your-lifetime-example.com/lifetimeapi/rest/v2`_
* **PropertiesAPI:** _`https://your-lifetime-example.com/PropertiesAPI/rest/v1`_

#### 4.2.2  Site Properties configuration

Access the Site Properties tab and configure the following properties:

* **Feature_UseTriggerManifest:** _True_
* **ServiceAccountToken:** &lt;your LifeTime authentication token&gt;

> **Note:**<br/>
>
> If you wish to tag changes on the fly when triggering the pipeline, the Service Account must have at least Change & Deploy permissions on the target environment.<br/>

#### 4.3 Configure the plugin Default Server Settings

After the plugin is successfully published and configured in the LifeTime environment, select **Configure Triggers** from the plugin landing page in LifeTime and configure the following parameters:

* **Source Environment:** _&lt;Select your OutSystems Development environment.&gt;_
* **Pipeline Server Type:** _Azure DevOps_
* **Pipeline Server Address:** _&lt;Your Azure Pipeline instance address, including organization and project. For example, `https://dev.azure.com/{organization}/{project}`.&gt;_
* **Pipeline Server Credentials:** _&lt;Credentials of an Azure user account with enough permissions for running pipeline jobs. You must use a [personal access token](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops) instead of a regular password for this authentication, and we recommended creating a dedicated user for this purpose.&gt;_

#### 4.4 Create a Pipeline trigger

Pipeline triggers can be configured by providing the following data:

* **Pipeline:** _&lt;Unique name that identifies the pipeline in Azure DevOps&gt;_
* **Pipeline Type:** _&lt;Select YAML for Multistage Pipeline&gt;_
* **Environment Labels (optional):** _&lt;Define the label assigned to each OutSystems environment along the deployment path&gt;_
* **Applications:** _&lt;List of LifeTime applications that will trigger the CI/CD pipeline, identifying which ones are Test applications&gt;_

After the Trigger Pipeline plugin is properly configured, the dashboard screen will show the list of pipeline triggers, along with the current versions in Development of the LifeTime applications defined for each pipeline scope.  

Once there are new application versions available for triggering a pipeline, a button is shown that allows running the pipeline on-demand without the need to log in to Azure.