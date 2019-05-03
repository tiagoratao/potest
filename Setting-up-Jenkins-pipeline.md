# Setting up a Jenkins pipeline

This document will go through the basics of setting up a Pipeline in Jenkins for your OutSystems applications. You're free to customize and alter it after it or extend it as you see fit.

This pipeline assumes you use 5 Environments: Development, Regression, Quality Assurance, Pre-Production and Production.

If you don't have as many environments, you can trim the parameters (and the Jenkinsfile) to suit your environment size. The major requirements are 3 environments: 1 DEV environment, 1 Regression environment, 1 Quality Assurance / Production. We recommend 5, to allow for other test cycles to be plugged to the pipeline as well.

We also chose to use parameterized builds in order to have more flexibility on the pipelines. This means we can have a single Jenkinsfile for all the pipelines but also tune it based on the parameters. It will also allow for the OutSystems LifeTime trigger plugin to trigger pipelines with LifeTime information about the apps to deploy. Although the initial setup is higher, future pipelines can be created by cloning from one existing one and changing the parameter values.

You can, of course, customize the Jenkinsfile to have hardcoded values and ignore the pipeline parameterization, creating one Jenkinsfile per pipeline and use your code repository to control Jenkinsfile's versioning and branching.

## Pre-Requirements before creating the pipeline

### Creating the Authentication Token for LifeTime

To automate the interaction between Jenkins and OutSystems LifeTime you'll need to create a service account and an access token. To do that, access your LifeTime environment and go to **User Management** > **Service Accounts** (e.g. <https://LT_URL/lifetime/ServiceAccounts_List.aspx>).

Press the **New Service Account** link under Service Accounts. Name it something you recognize. The username can be anything, since it won't be used.

Select a role that is able to deploy to all the environments you plan on extending the pipeline. It needs **Change & Deploy** in those environments.

Press create and save the token it will generate.

On Jenkins, go to the **Credentials** menu and add a new credential to the domain you want (e.g. Global credentials). The credentials you're going to add will have the following format:

----------
**Kind:** Secret Text

**Scope:** Global

**Secret:** \<The Token from LifeTime>

**ID:** \<The name of the Token to be used in the pipeline parameter>

**Description:** Token for OutSystems LifeTime

----------

### Creating GIT repository for Jenkinsfile

You will want to create a repository to store your custom Jenkinsfile. In this documentation we will use GitHub but you can use any SCM provided you know how to use it.

After you create the repository and commit the Jenkinsfile, you'll need to create an access key so that Jenkins can clone the repository. This is needed if your repository is not public.

To do that, go to <https://github.com/ORGANIZATION/REPO_NAME/settings/keys> and create a key pair.

On Jenkins, under **Credentials**, create a new credential, similar to the LifeTime Token, with the following attributes:

----------
**Kind:** SSH Username with private key

**Scope:** Global

**ID:** \<The name of the key to be used in the pipeline, under SCM>

**Description:** Jenkins key for XYZ repo

**Username:** \<Irrelevant, it won't be used>

**Private Key:** Select **Enter directly** and paste in the private key.

**Passphrase:** \<If it's being used, insert here the passphrase>

----------

### Installing the OutSystems LifeTime Trigger plugin

The OutSystems Forge link for [Trigger Pipeline](https://www.outsystems.com/forge/component-overview/5670/trigger-pipeline), where you can download it.

You can follow this [guide](https://success.outsystems.com/Documentation/11/Getting_Started/Use_a_Forge_Component_Made_by_the_Community) on how to install in your **LifeTime** environment. Note: you only need to install it, you don't need to manage the dependencies.

### Installing the CICD Probe and BDD Framework

The OutSystems Forge URL for [BDD Framework](https://www.outsystems.com/forge/component-overview/1201/bddframework) and [CICD Probe](https://TODO), where you can download them.

You can follow this [guide](https://success.outsystems.com/Documentation/11/Getting_Started/Use_a_Forge_Component_Made_by_the_Community) on how to install in your **Regression** environment. Note: you only need to install it, you don't need to manage the dependencies.

## Creating the pipeline

Access to your Jenkins platform and, on the left menu, select **New Item**.

![New Item](images/new_item.png)

A window will appear. Choose a name for the pipeline. You can you other types of pipeline if you feel more confortable but since the goal here is simplicity, we will use the **Pipeline** type. Then click OK

![Pipeline Type](images/pipeline_type.png)

Next, you'll see the pipeline configuration page. In the following steps we will describe the basic configuration you'll need to do to work with the base Jenkinsfile provided in the OutSystems Pipeline repository.

### Parameters

Under **General**, check the box **This project is parameterized**.

![Activating parameters](images/param_project.png)

This will show a button to add the parameters:

![Adding parameters](images/param_check.png)

Click on the **Add Parameter** and select the parameter type based on the following:

----------
**Name:** AppScope

**Type:** String

**Default Value:** <App name or comma separated list of apps to deploy, without the tests>

**Description:** Name of the App(s) without the tests, to deploy. If you add more than one, use a comma to separate them. Example: App1,App2 With Spaces,App3_With_Underscores

----------
**Name:** AppWithTests

**Type:** String

**Default Value:** <App name or comma separated list of apps to deploy, **with** the tests>

**Description:** Name of the App(s) with the tests, to deploy. If you add more than one, use a comma to separate them. Example: App1,App2 With Spaces,App3_With_Underscores

----------
**Name:** LTApiVersion

**Type:** String

**Default Value:** <depends on your LT OutSystems platform. If version <= 10, use 1, if version >= 11, use 2>

**Description:** LifeTime API version number. If version <= 10, use 1, if version >= 11, use 2.

----------
**Name:** LTUrl

**Type:** String

**Default Value:** \<URL for LT>

**Description:** URL for LifeTime, without the API endpoint and the trailing slash and the HTTPS protocol (<https://>)

----------

**Name:** LTToken

**Type:** Credentials

**Credential type:** Secret Text

**Required:** True

**Default Value:** \<You can either create here the credential for the LT Token or select on previously created>

**Description:** Token for LifeTime Access.

----------
**Name:** DevEnv

**Type:** String

**Default Value:** \<Name of the Dev Environment>

**Description:** Name of the development environment, as shown in LifeTime

----------
**Name:** RegEnv

**Type:** String

**Default Value:** \<Name of the Reg Environment>

**Description:** Name of the regression environment, as shown in LifeTime

----------
**Name:** QAEnv

**Type:** String

**Default Value:** \<Name of the QA Environment>

**Description:** Name of the quality assurance environment, as shown in LifeTime

----------
**Name:** PpEnv

**Type:** String

**Default Value:** \<Name of the PP Environment>

**Description:** Name of the pre-production environment, as shown in LifeTime

----------
**Name:** PrdEnv

**Type:** String

**Default Value:** \<Name of the PRD Environment>

**Description:** Name of the production environment, as shown in LifeTime

----------
**Name:** ProbeUrl

**Type:** String

**Default Value:** \<URL for the environment where the CICD Probe is running (e.g. Regression)>

**Description:** URL for the environment hosting the CICD Probe.

----------
**Name:** BddUrl

**Type:** String

**Default Value:** \<URL for the environment where the BDD Framework is running (e.g. Regression)>

**Description:** URL for the environment hosting the BDD Framework.

----------

### Build triggers

To allow Jenkins to be triggered by OutSystems LifeTime, you need to set a token. Go to the build trigger section and check **Trigger builds remotely (e.g., from scripts)**:

![Build Trigger](images/build_trigger.png)

If this option does not show, make sure you have the **Build Authorization Token Root** plugin installed.

A box will appear for you to insert the token. Generate a token and paste it there. Currently, this token needs to be the same for all pipelines but we are working to change it in the future.

![Build Token](images/build_token.png)

### Source Control

Under **Pipeline**, select **Pipeline script from SCM**. Then select **GIT** as the SCM and configure the repository. Insert the .git link to your repo, where you have stored your Jenkinsfile. If you haven't setup a personal repository, now would be a good time to do it and clone the Jenkinsfile provided in the OutSystems Pipeline repository as a baseline. Since this will use the GIT procotocol, don't forget to put it as `git@<url>.git`.

As for the credentials, select the previously created GIT credentials.

Select the branch you want to use (e.g. master). Avoid using **any** or else you might have unexpected results where pipelines are using the wrong Jenkinsfile.

Finally, set up the **Script Path** with the path from the root of the repository until you read your Jenkinsfile. If the Jenkinsfile is on the root itself, just type in the name of the Jenkinsfile.

Click **Save** to finish the pipeline configuration

## Setting up the OutSystems LifeTime trigger

Note: This is the process described for v1.0.

Open the trigger plugin in your LifeTime environment (e.g. <https://LTURL/TriggerPipeline/>). Click on **Configure Triggers**, under Trigger Dashboard.

On there, select the source environment (i.e. your Development environment).

Set up the Pipeline Server Endpoint with the URL for Jenkins and the **Build Token** you used when you defined the pipeline. The URL Endpoint will something like <https://JENKINS_URL/buildByToken/buildWithParameters?job={PipelineName}&token=TOKEN>. As you can see, the URL is parameterized with the variable `{PipelineName}`. This means your pipelines configured on the trigger must have the same name as the ones present on Jenkins. This way, LifeTime will trigger the right pipeline.

Save the configurations.

To create a new pipeline, press the **New Pipeline Trigger** link under Configuration. Here you'll name your pipeline (Remember: use the **same** name as the pipeline on Jenkins) and select which apps belong to that pipeline, including the ones with the tests.

In the next release of the plugin, the UI will split what's test apps from the regular apps and allow for more customization.

Press **save** and the pipeline will be saved and the configuration is complete.

## Running the pipeline

To run the pipeline, you only need to [tag the applications](https://success.outsystems.com/Documentation/11/Managing_the_Applications_Lifecycle/Deploy_Applications/Tag_a_Version), on OutSystems LifeTime, you want to deploy, then go to the Trigger Plugin and press the **Trigger Pipeline** button. This button will only be available if you have applications ready to be deployed. If not it will be grayed out.

Press the button and the pipeline will be lauched on Jenkins.

## Creating other pipelines

On the Jenkins dashboard, click **New Item**. Set the **Name** and use the **Copy from** to copy from another pipeline. Then click OK.

Change the values for all the parameters, to match your application details. Press save to generate the pipeline.