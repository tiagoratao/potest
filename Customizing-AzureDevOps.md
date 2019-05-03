# Customizing the Azure DevOps pipeline

## Adding the Slack package

\<TODO>

* **SlackChannels**: (Optional if you use the slack plugin) Name of the Slack Channel(s) you wish to send notifications. For multiple channels, use a comma-separated list. *Example:* Channel1,Channel-2.
* **SlackHook**: (Optional if you use the slack plugin) Slack hook to make API calls. **Important**: Set this as a secret type, to avoid having it shown on the logs.

* (Optional if you use the slack plugin) One task (either **Bash** for Linux agents or **PowerShell** for Windows agents)
  * Linux:
    * Script Path: `$(System.DefaultWorkingDirectory)/$(Release.PrimaryArtifactSourceAlias)/cd_pipelines/azure_devops/send_notifications_slack.sh`
    * Arguments: `-e "$(PythonEnvName)" -a "$(ArtifactsFolder)" -s $(SlackHook) -c "$(SlackChannels)" -p "$(PipelineType)" -j "$(JobName)" -d $(DashBoardUrl)`
    * Working Directory: `$(Release.PrimaryArtifactSourceAlias)`
  * Windows:
    * Script Path: `$(System.DefaultWorkingDirectory)/$(Release.PrimaryArtifactSourceAlias)/cd_pipelines/azure_devops/send_notifications_slack.ps1`
    * Arguments: `-PythonEnv "$(PythonEnvName)" -ArtifactDir "$(ArtifactsFolder)" -SlackHook $(SlackHook) -SlackChannels "$(SlackChannels)" -PipelineType "$(PipelineType)" -JobName "$(JobName)" -DashboardUrl $(DashBoardUrl)`
    * Working Directory: `$(Release.PrimaryArtifactSourceAlias)`

## Adding one environment