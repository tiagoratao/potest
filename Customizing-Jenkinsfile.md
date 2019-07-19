# Customizing your Jenkinsfile

## Adding one environment

On the Jenkinsfile, one environment corresponds to a Stage in the pipeline, for simplicity's sake. The stages have the following configuration:

~~~~~groovy
stage('Deploy to <EnvironmentName> Environment') {
    steps {
        withPythonEnv('python') {
            echo 'Deploying latest application tags to <EnvironmentName>...'
            powershell "python outsystems-pipeline.pipeline.deploy_latest_tags_to_target_env.py --artifacts \"${env.ArtifactsFolder}\" --lt_url ${env.LifeTimeEnvironmentURL} --lt_token ${env.AuthorizationToken} --lt_api_version ${env.LifeTimeAPIVersion} --source_env \"<SRC_ENV_NAME_PARAM>\" --destination_env \"<DEST_ENV_NAME_PARAM>\" --app_list \"${params.AppScope}\""
        }
    }
    post {
        always {
            dir ("${env.ArtifactsFolder}") {
                archiveArtifacts artifacts: "*_data/*.cache", onlyIfSuccessful: true
            }
        }
        failure {
            dir ("${env.ArtifactsFolder}") {
                archiveArtifacts artifacts: "DeploymentConflicts"
            }
        }
    }
}
~~~~~

You'll need the following variables (some you should already have from previous steps):

* Artifact Directory: `${env.ArtifactsFolder}`
* LifeTime URL: `{env.LifeTimeEnvironmentURL}`
* LifeTime Token: `${env.AuthorizationToken}`
* LifeTime API Version: `${env.LifeTimeAPIVersion}`
* Source Environment Name: The name of the environment you want to deploy *from*
* Destination Environment Name: The name of the environment you want to deploy *to*
* App List: `${params.AppScope}`
  * if you want to deploy with the tests apps, change it to `${env.ApplicationsWithTests}`

The **Post** step will define what Jenkins will do in case of failure / success.

The `always` portion will always run. In this step we archive all the cache files if the stage was successfull. You can remove the `onlyIfSuccessful: true` or change it to `false` if you want to save the `.cache` files in case of errors.

The `failure` will only run if there's a failure in the stage. In this case we are only concerned with Deployment Conflicts, so we archive the file generated when there's one.

The file archiving allows us to see the files on the job executions, on Jenkins.

## Using the Slack package

**Important:** Please note that the Slack Package is not included with the OutSystems Python package. You'll need to add it to your personal Git repository, by cloning that part of the repo.

Before you start using the Slack package, you'll need to configure the following parameters on your pipeline:

----------
**Name:** SlackChannelList

**Type:** String

**Default Value:** \<Channel or list of channels>

**Description:** Name of the Slack Channel you wish to send notifications. For more than one, use a comma-separated list. E.g.: channel1,channel-2,channel-num-3

----------

**Name:** SlackHook

**Type:** Credentials

**Credential type:** Secret Text

**Required:** True

**Default Value:** \<You can either create here the Slack Hook or select on previously created>

**Description:** Slack Hook for API requests

----------

For more information on how to create a Slack Web Hook see the [documentation](https://api.slack.com/incoming-webhooks) and / or use the [app](https://<your_org>.slack.com/apps/A0F7XDUAZ-incoming-webhooks).

Refer back to [Creating the Authentication Token for LifeTime](Setting-up-Jenkins-pipeline#creating-the-authentication-token-for-lifetime) for more information on how to create a credential on Jenkins.

On your Jenkinsfile, under `environment` you can add the new variables:

~~~~groovy
environment {
    ...

    // Fetch the Slack Hook from the credentials
    SlackHook = credentials("${params.SlackHook}")
    // Slack channel list
    SlackChannels = "${params.SlackChannelList}"

    ...
~~~~

Usually, Slack is used to report the status of a stage. You can use the **Post** step inside the stage to accomplish this. To use the Slack package you can use:

### Generic errors

~~~~groovy
failure {
    ...

    withPythonEnv('python') {
        echo 'Sending failure message to slack...'
        powershell "python .\\outsystems_integrations\\slack\\send_pipeline_status_to_slack.py --artifacts \"${env.ArtifactsFolder}\" --slack_hook ${env.SlackHook} --slack_channel \"${env.SlackChannels}\" --pipeline jenkins --status false --title \"*Pipeline Error: ${env.JOB_NAME}*\" --message \"<message>\""
    }

    ...
}
~~~~

### Deployment Conflict errors

~~~~groovy
failure {
    ...

    withPythonEnv('python') {
        echo 'Sending failure message to slack...'
        powershell "python .\\outsystems_integrations\\slack\\send_pipeline_status_to_slack.py --artifacts \"${env.ArtifactsFolder}\" --slack_hook ${env.SlackHook} --error_in_file DeploymentConflicts --slack_channel \"${env.SlackChannels}\" --pipeline jenkins --status false --title \"*Pipeline Error: ${env.JOB_NAME}*\" --message \"Failed trying to deploys apps from *<SRC ENV>* to *<DEST ENV>*.\""
    }

    ...
}
~~~~

### Test results notification

~~~~groovy
always {
    ...

    withPythonEnv('python') {
        echo 'Sending test results to slack...'
        powershell "python .\\outsystems_integrations\\slack\\send_test_results_to_slack.py --artifacts \"${env.ArtifactsFolder}\" --slack_hook ${env.SlackHook} --slack_channel \"${env.SlackChannels}\" --pipeline jenkins --job_name \"${env.JOB_NAME}\" --job_dashboard_url ${env.RUN_DISPLAY_URL}"
    }

    ...
}
~~~~

### Waiting to proceed notification

~~~~groovy
steps {
    ...

    withPythonEnv('python') {
        echo 'Sending waiting message to slack...'
        powershell "python .\\outsystems_integrations\\slack\\send_pipeline_status_to_slack.py --artifacts \"${env.ArtifactsFolder}\" --slack_hook ${env.SlackHook} --slack_channel \"${env.SlackChannels}\" --pipeline jenkins --status true --title \"*Pipeline Waiting: ${env.JOB_NAME}*\" --message \"Pipeline needs your input to progress to *<ENVIRONMENT NAME>*.\n\nGo here to confirm: ${env.RUN_DISPLAY_URL}\""

    ...
}
~~~~
