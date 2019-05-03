# Setting up an Azure DevOps pipeline (Beta)

This guide assumes your factory is composed of 5 environments: Development, Regression, Quality Assurance, Pre-Production and Production. If that differs from your environment, just add/remove the environments (in the pipeline) that are not needed.

To create a pipeline on Azure DevOps, you can use the scripts defined under **cd_pipelines > azure_devops**. You can use either Linux or Windows as your agents. If you use Linux, use the *.sh scripts, if you use Windows use the *.ps1 scripts.

**Important!**

You should clone the folder *cd_pipelines* > *azure_devops* to your local repository. You can clone it to GitHub or Azure DevOps repository. If you're using GitHub, you'll need to create a connection to GitHub on Azure DevOps.

The Release pipeline artifact will be the direct connection to the repository. Since you don't need to compile code, you don't need a Build Pipeline.
If you use more than one artifact make sure that the artifact with the pipeline scripts is the **primary** artifact!

In order to run it on Azure DevOps you'll create either **Bash** tasks (for Linux) or **PowerShell** tasks (for Windows).

Since Azure DevOps doesn't support Release Pipelines as Code yet, we can't give you a working example, so the way you set up your pipeline is left as an exercise. But there are still parameters you need to setup. Under the variable tab, you can choose either to use Pipeline variables or Variable groups and link those to specific environments. Again, is left to your choice at this moment. We will describe the process with *Pipeline variables*.

## Variables

The following variables are to be set for the Release Scope:

* **AppScope**: Name of the App(s) without the tests, to deploy. If you add more than one, use a comma to separate them. *Example:* App1,App2 With Spaces,App3_With_Underscores.
* **AppWithTests**: Name of the App(s) without the tests, to deploy. If you add more than one, use a comma to separate them. *Example:* App1,App2 With Spaces,App3_With_Underscores.
* **ArtifactsFolder**: Name of the Artifacts folder, where the pipeline will store the cache files between Jobs.
* **BDDFrameworkURL**: URL of the environment, without the API part, of the BDD Framework (e.g. `https://<regression_hostname>`).
* **CICDProbeURL**: URL of the environment, without the API part, of the CICD Probe (e.g. `https://<regression_hostname>`).
* **DashBoardUrl**: (Optional if you use the slack plugin) URL for the dashboard link you'll receive in the slack notification.
* **DeployMsg**: Message that will be written on the deployment plan in LifeTime.
* **JobName**: (Optional if you use the slack plugin) Name of the Pipeline you're deploying.
* **LTAPIVersion**: LifeTime API version number. If version <= 10, use 1, if version >= 11, use 2.
* **LTToken**: Token that you'll use to interact with LifeTime's API. It should be created, on LifeTime, as a Service Account and given enough priviledges to deploy the code until production. **Important**: Set this as a secret type, to avoid having it shown on the logs.
* **LTURL**: URL for LifeTime, without the API endpoint and the trailing slash and the HTTPS protocol (`https://`).
* **PipelineType**: (Optional if you use the slack plugin) Since this Azure DevOps, set it as *azure*.
* **PythonEnvName**: Name of the Python Environment where the pipeline dependencies will be installed. Example: OSPipeline
* **DevEnv**: Name of the development environment, as shown in LifeTime.
* **RegEnv**: Name of the regression environment, as shown in LifeTime.
* **QAEnv**: Name of the quality assurance environment, as shown in LifeTime.
* **PPEnv**: Name of the pre-production environment, as shown in LifeTime.
* **PRDEnv**: Name of the production environment, as shown in LifeTime.

## Environments

The division between Environments is up to you (at this moment) but we suggest a division similar to the Jenkins one:

* Install Python Dependencies and create Artifact directory
* Get Latest Applications and Environments from LifeTime
* Deploy tags to Regression Environment
* Run Regression tests on the Regression Environment
* Deploy tags to Quality Assurance Environment
* Deploy tags to PP Environment
* Deploy tags to PRD Environment

## Jobs & Tasks

Since Azure DevOps' Python tasks don't support Python Environments, we need to wrap our Python calls in scripts. The following Jobs will need to be setup on the pipeline:

### Install Python Dependencies and create Artifact directory

#### Agent Job

* Use either:
  * Linux agent pool
  * Windows Agent pool

##### Task

* Use either:
  * **Bash** for Linux agents
    * Script path: `$(System.DefaultWorkingDirectory)/$(Release.PrimaryArtifactSourceAlias)/cd_pipelines/azure_devops/install_dependencies.sh`
    * Arguments: `-e "$(PythonEnvName)" -a "$(ArtifactsFolder)"`
    * Working Directory: `$(Release.PrimaryArtifactSourceAlias)`
  * **PowerShell** for Windows agents
    * Script Path: `$(System.DefaultWorkingDirectory)/$(Release.PrimaryArtifactSourceAlias)/cd_pipelines/azure_devops/install_dependencies.ps1`
    * Arguments: `-PythonEnv "$(PythonEnvName)" -ArtifactDir "$(ArtifactsFolder)"`
    * Working Directory: `$(Release.PrimaryArtifactSourceAlias)`

### Get Latest Applications and Environments from LifeTime

#### Agent Job

* Use either:
  * Linux agent pool
  * Windows Agent pool

##### Task

* Use either:
  * **Bash** for Linux agents
    * Script Path: `$(System.DefaultWorkingDirectory)/$(Release.PrimaryArtifactSourceAlias)/cd_pipelines/azure_devops/fetch_lt_data.sh`
    * Arguments: `-e "$(PythonEnvName)" -a "$(ArtifactsFolder)" -u $(LTURL) -t $(LTToken) -v $(LTAPIVersion)`
    * Working Directory: `$(Release.PrimaryArtifactSourceAlias)`
  * **PowerShell** for Windows agents
    * Script Path: `$(System.DefaultWorkingDirectory)/$(Release.PrimaryArtifactSourceAlias)/cd_pipelines/azure_devops/fetch_lt_data.ps1`
    * Arguments: `-PythonEnv "$(PythonEnvName)" -ArtifactDir "$(ArtifactsFolder)" -LifeTimeUrl $(LTURL) -LifeTimeToken $(LTToken) -LifeTimeApi $(LTAPIVersion)`
    * Working Directory: `$(Release.PrimaryArtifactSourceAlias)`

### Deploy tags to Regression Environment

#### Agent Job

* Use either:
  * Linux agent pool
  * Windows Agent pool

##### Task

* Use either:
  * **Bash** for Linux agents
    * Script Path: `$(System.DefaultWorkingDirectory)/$(Release.PrimaryArtifactSourceAlias)/cd_pipelines/azure_devops/deploy_apps_to_env.sh`
    * Arguments: `-e "$(PythonEnvName)" -a "$(ArtifactsFolder)" -u $(LTURL) -t $(LTToken) -v $(LTAPIVersion) -s "$(DevEnv)" -d "$(RegEnv)" -l "$(AppScope),$(AppWithTests)" -m "$(DeployMsg)"`
    * Working Directory: `$(Release.PrimaryArtifactSourceAlias)`
  * **PowerShell** for Windows agents
    * Script Path: `$(System.DefaultWorkingDirectory)/$(Release.PrimaryArtifactSourceAlias)/cd_pipelines/azure_devops/deploy_apps_to_env.ps1`
    * Arguments: `-PythonEnv "$(PythonEnvName)" -ArtifactDir "$(ArtifactsFolder)" -LifeTimeUrl $(LTURL) -LifeTimeToken $(LTToken) -LifeTimeApi $(LTAPIVersion) -SourceEnv "$(DevEnv)" -DestEnv "$(RegEnv)" -AppList "$(AppScope),$(AppWithTests)" -DeployMsg "$(DeployMsg)"`
    * Working Directory: `$(Release.PrimaryArtifactSourceAlias)`

### Run Regression tests on the Regression Environment

#### Agent Job

* Use either:
  * Linux agent pool
  * Windows Agent pool

##### Task

* Use either:
  * **Bash** for Linux agents
    * Script Path: `$(System.DefaultWorkingDirectory)/$(Release.PrimaryArtifactSourceAlias)/cd_pipelines/azure_devops/build_test_endpoints.sh`
    * Arguments: `-e "$(PythonEnvName)" -a "$(ArtifactsFolder)" -l "$(AppWithTests)" -c $(CICDProbeURL) -b $(BDDFrameworkURL)`
    * Working Directory: `$(Release.PrimaryArtifactSourceAlias)`
  * **PowerShell** for Windows agents
    * Script Path: `$(System.DefaultWorkingDirectory)/$(Release.PrimaryArtifactSourceAlias)/cd_pipelines/azure_devops/build_test_endpoints.ps1`
    * Arguments: `-PythonEnv "$(PythonEnvName)" -ArtifactDir "$(ArtifactsFolder)" -AppList "$(AppWithTests)" -BddUrl $(BDDFrameworkURL) -CicdUrl $(CICDProbeURL)`
    * Working Directory: `$(Release.PrimaryArtifactSourceAlias)`

##### Task

* Use either:
  * **Bash** for Linux agents
    * Script Path: `$(System.DefaultWorkingDirectory)/$(Release.PrimaryArtifactSourceAlias)/cd_pipelines/azure_devops/running_test_endpoints.sh`
    * Arguments: `-e "$(PythonEnvName)" -a "$(ArtifactsFolder)"`
    * Working Directory: `$(Release.PrimaryArtifactSourceAlias)`
  * **PowerShell** for Windows agents
    * Script Path: `$(System.DefaultWorkingDirectory)/$(Release.PrimaryArtifactSourceAlias)/cd_pipelines/azure_devops/running_test_endpoints.ps1`
    * Arguments: `-PythonEnv "$(PythonEnvName)" -ArtifactDir "$(ArtifactsFolder)"`
    * Working Directory: `$(Release.PrimaryArtifactSourceAlias)`

#### Task

* Use: 
  * **Publish Test Results**
    * Test result format: `JUnit`
    * Test results files: `**/junit-result.xml`
    * Search Folder: `$(System.DefaultWorkingDirectory)/$(Release.PrimaryArtifactSourceAlias)/$(ArtifactsFolder)`
    * Fail if there are test failures: `true`

### Deploy tags to Quality Assurance Environment

#### Agentless Job

##### Task

* Use:
  * **Manual Intervention**
    * Instructions: `Do you approve deployment to QA?`
    * Notify Users: `<user list>`

#### Agent Job

* Use either:
  * Linux agent pool
  * Windows Agent pool

##### Task

* Use either:
  * **Bash** for Linux agents
    * Script Path: `$(System.DefaultWorkingDirectory)/$(Release.PrimaryArtifactSourceAlias)/cd_pipelines/azure_devops/deploy_apps_to_env.sh`
    * Arguments: `-e "$(PythonEnvName)" -a "$(ArtifactsFolder)" -u $(LTURL) -t $(LTToken) -v $(LTAPIVersion) -s "$(RegEnv)" -d "$(QAEnv)" -l "$(AppScope),$(AppWithTests)" -m "$(DeployMsg)"`
    * Working Directory: `$(Release.PrimaryArtifactSourceAlias)`
  * **PowerShell** for Windows agents
    * Script Path: `$(System.DefaultWorkingDirectory)/$(Release.PrimaryArtifactSourceAlias)/cd_pipelines/azure_devops/deploy_apps_to_env.ps1`
    * Arguments: `-PythonEnv "$(PythonEnvName)" -ArtifactDir "$(ArtifactsFolder)" -LifeTimeUrl $(LTURL) -LifeTimeToken $(LTToken) -LifeTimeApi $(LTAPIVersion) -SourceEnv "$(RegEnv)" -DestEnv "$(QAEnv)" -AppList "$(AppScope),$(AppWithTests)" -DeployMsg "$(DeployMsg)"`
    * Working Directory: `$(Release.PrimaryArtifactSourceAlias)`

### Deploy tags to PP Environment

#### Agentless Job

##### Task

* Use:
  * **Manual Intervention**
    * Instructions: `Do you approve deployment to PP and Production?`
    * Notify Users: `<user list>`

#### Agent Job

* Use either:
  * Linux agent pool
  * Windows Agent pool

##### Task

* Use either:
  * **Bash** for Linux agents
    * Script Path: `$(System.DefaultWorkingDirectory)/$(Release.PrimaryArtifactSourceAlias)/cd_pipelines/azure_devops/deploy_apps_to_env.sh`
    * Arguments: `-e "$(PythonEnvName)" -a "$(ArtifactsFolder)" -u $(LTURL) -t $(LTToken) -v $(LTAPIVersion) -s "$(QAEnv)" -d "$(PPEnv)" -l "$(AppScope),$(AppWithTests)" -m "$(DeployMsg)"`
    * Working Directory: `$(Release.PrimaryArtifactSourceAlias)`
  * **PowerShell** for Windows agents
    * Script Path: `$(System.DefaultWorkingDirectory)/$(Release.PrimaryArtifactSourceAlias)/cd_pipelines/azure_devops/deploy_apps_to_env.ps1`
    * Arguments: `-PythonEnv "$(PythonEnvName)" -ArtifactDir "$(ArtifactsFolder)" -LifeTimeUrl $(LTURL) -LifeTimeToken $(LTToken) -LifeTimeApi $(LTAPIVersion) -SourceEnv "$(QAEnv)" -DestEnv "$(PPEnv)" -AppList "$(AppScope),$(AppWithTests)" -DeployMsg "$(DeployMsg)"`
    * Working Directory: `$(Release.PrimaryArtifactSourceAlias)`

### Deploy tags to PRD Environment

#### Agent Job

* Use either:
  * Linux agent pool
  * Windows Agent pool

##### Task

* Use either:
  * **Bash** for Linux agents
    * Script Path: `$(System.DefaultWorkingDirectory)/$(Release.PrimaryArtifactSourceAlias)/cd_pipelines/azure_devops/deploy_apps_to_env.sh`
    * Arguments: `-e "$(PythonEnvName)" -a "$(ArtifactsFolder)" -u $(LTURL) -t $(LTToken) -v $(LTAPIVersion) -s "$(PPEnv)" -d "$(PRDEnv)" -l "$(AppScope),$(AppWithTests)" -m "$(DeployMsg)"`
    * Working Directory: `$(Release.PrimaryArtifactSourceAlias)`
  * **PowerShell** for Windows agents
    * Script Path: `$(System.DefaultWorkingDirectory)/$(Release.PrimaryArtifactSourceAlias)/cd_pipelines/azure_devops/deploy_apps_to_env.ps1`
    * Arguments: `-PythonEnv "$(PythonEnvName)" -ArtifactDir "$(ArtifactsFolder)" -LifeTimeUrl $(LTURL) -LifeTimeToken $(LTToken) -LifeTimeApi $(LTAPIVersion) -SourceEnv "$(PPEnv)" -DestEnv "$(PRDEnv)" -AppList "$(AppScope),$(AppWithTests)" -DeployMsg "$(DeployMsg)"`
    * Working Directory: `$(Release.PrimaryArtifactSourceAlias)`