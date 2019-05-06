# Customizing your Jenkinsfile

## Adding one environment

On the Jenkinsfile, one environment corresponds to a Stage in the pipeline. The stages have the following configuration:

~~~~~groovy
stage('Deploy to <EnvironmentName> Environment') {
    steps {
        withPythonEnv('python') {
            echo 'Deploying latest application tags to <EnvironmentName>...'
            powershell "python .\\outsystems\\pipeline\\deploy_latest_tags_to_target_env.py --artifacts \"${env.ArtifactsFolder}\" --lt_url ${env.LifeTimeEnvironmentURL} --lt_token ${env.AuthorizationToken} --lt_api_version ${env.LifeTimeAPIVersion} --source_env \"<SRC_ENV_NAME_PARAM>\" --destination_env \"<DEST_ENV_NAME_PARAM>\" --app_list \"${params.AppScope}\""
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

The **Post** part will define what Jenkins will do in case of failure / success.

The `always` portion will always run. In this step we archive all the cache files if the stage was successfull. You can remove the `onlyIfSuccessful: true` or change it to `false` if you want to save the `.cache` files in case of errors.

The `failure` will only run if there's a failure in the stage. In this case we are only concerned with Deployment Conflicts, so we archive the file generated when there's one.

The file archiving allows us to see the files on the job executions, on Jenkins.

## Adding the Slack package

\<TODO>