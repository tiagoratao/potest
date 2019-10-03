The following reference describes the syntax and arguments of the Python scripts included in the package `outsystems.pipeline`:

* [outsystems.pipeline.fetch_lifetime_data](#outsystems.pipeline.fetch_lifetime_data)
* [outsystems.pipeline.deploy_latest_tags_to_target_env](#outsystems.pipeline.deploy_latest_tags_to_target_env)
* [outsystems.pipeline.generate_unit_testing_assembly](#outsystems.pipeline.generate_unit_testing_assembly)
* [outsystems.pipeline.evaluate_test_results](#outsystems.pipeline.evaluate_test_results)

## outsystems.pipeline.fetch_lifetime_data

Obtains the full list of environments and applications on LifeTime and caches
the retrieved information in the artifacts folder.

```
fetch_lifetime_data.py [-h] [-a ARTIFACTS]
                            -u LT_URL
                            -t LT_TOKEN
                            [-v LT_API_VERSION]
                            [-e LT_ENDPOINT]
```

### Arguments

* **-h, --help**

  Show this help message and exit.

* **-a ARTIFACTS, --artifacts ARTIFACTS**

  (Optional) Name of the artifacts folder. Default: "Artifacts"

* **-u LT_URL, --lt_url LT_URL**

  URL of the LifeTime environment, without the API endpoint. Example: "https://<lifetime_host>"

* **-t LT_TOKEN, --lt_token LT_TOKEN**

  LifeTime authentication token used in the API requests.

* **-v LT_API_VERSION, --lt_api_version LT_API_VERSION**

  (Optional) LifeTime Deployment API version to use. Use 2 for OutSystems 11 and above and 1 for OutSystems 10. Default: 2

* **-e LT_ENDPOINT, --lt_endpoint LT_ENDPOINT**

  (Optional) Overrides the default LifeTime API endpoint, without the version. Default: "lifetimeapi/rest"

## outsystems.pipeline.deploy_latest_tags_to_target_env

Creates and runs a LifeTime deployment plan to deploy a set of applications
from the source environment to the destination environment.

The deployment plan only includes the latest tagged versions of the
applications from the source environment that do not already exist on the target
environment.

```
deploy_latest_tags_to_target_env.py [-h] [-a ARTIFACTS]
                                         -u LT_URL
                                         -t LT_TOKEN
                                         [-v LT_API_VERSION]
                                         [-e LT_ENDPOINT]
                                         -s SOURCE_ENV
                                         -d DESTINATION_ENV
                                         -l APP_LIST
                                         [-m DEPLOY_MSG]
```

### Arguments

* **-h, --help**

  Show this help message and exit.

* **-a ARTIFACTS, --artifacts ARTIFACTS**

  (Optional) Name of the artifacts folder. Default: "Artifacts"

* **-u LT_URL, --lt_url LT_URL**

  URL of the LifeTime environment, without the API endpoint. Example: "https://<lifetime_host>"

* **-t LT_TOKEN, --lt_token LT_TOKEN**

  LifeTime authentication token used in the API requests.

* **-v LT_API_VERSION, --lt_api_version LT_API_VERSION**

  (Optional) LifeTime Deployment API version to use. Use 2 for OutSystems 11 and above and 1 for OutSystems 10. Default: 2

* **-e LT_ENDPOINT, --lt_endpoint LT_ENDPOINT**

  (Optional) Overrides the default LifeTime API endpoint, without the version. Default: "lifetimeapi/rest"

* **-s SOURCE_ENV, --source_env SOURCE_ENV**

  Name of the source environment, as displayed on LifeTime. Example: "Development"

* **-d DESTINATION_ENV, --destination_env DESTINATION_ENV**

  Name of the destination environment where to deploy the apps, as displayed on LifeTime. Example: "Regression"

* **-l APP_LIST, --app_list APP_LIST**

  Comma-separated list app names to deploy. Example: "App1,App2 With Spaces,App3_With_Underscores"

* **-m DEPLOY_MSG, --deploy_msg DEPLOY_MSG**

  (Optional) Note to include on the deployment plan. Default: "Automated deploy via OutSystems Pipeline"

## outsystems.pipeline.generate_unit_testing_assembly

Obtains the list of BDD Framework endpoints for a set of applications and
stores the retrieved information in the artifacts folder.

```
generate_unit_testing_assembly.py [-h] [-a ARTIFACTS]
                                       -l APP_LIST
                                       --cicd_probe_env CICD_PROBE_ENV
                                       [--cicd_probe_api CICD_PROBE_API]
                                       [--cicd_probe_version CICD_PROBE_VERSION]
                                       --bdd_framework_env BDD_FRAMEWORK_ENV
                                       [--bdd_framework_api BDD_FRAMEWORK_API]
                                       [--bdd_framework_version BDD_FRAMEWORK_VERSION]
```

### Arguments

* **-h, --help**

  Show this help message and exit.

* **-a ARTIFACTS, --artifacts ARTIFACTS**

  (Optional) Name of the artifacts folder. Default: "Artifacts"

* **-l APP_LIST, --app_list APP_LIST**

  Comma-separated list app names to deploy. Example: "App1,App2 With Spaces,App3_With_Underscores"

* **--cicd_probe_env CICD_PROBE_ENV**

  URL of the CI/CD Probe, without the API endpoint. Example: "https://<host>"

* **--cicd_probe_api CICD_PROBE_API**

  (Optional) Overrides the default CI/CD Probe API endpoint, without the version. Default: "CI_CDProbe/rest"

* **--cicd_probe_version CICD_PROBE_VERSION**

  (Optional) CI/CD Probe API version to use. Default: 1

* **--bdd_framework_env BDD_FRAMEWORK_ENV**

  URL of the BDD Framework, without the API endpoint. Example: "https://<host>"

* **--bdd_framework_api BDD_FRAMEWORK_API**

  (Optional) Overrides the default BDD Framework API endpoint, without the version. Default: "BDDFramework/rest"

* **--bdd_framework_version BDD_FRAMEWORK_VERSION**

  (Optional) BDD Framework API version to use. Default: 1

## outsystems.pipeline.evaluate_test_results

Builds and runs a test suite for the BDD Framework endpoints stored in the
artifacts folder and generates a JUnit XML test report in the artifacts folder

```
evaluate_test_results.py [-h] [-a ARTIFACTS]
```

### Arguments

* **-h, --help**

  Show this help message and exit.

* **-a ARTIFACTS, --artifacts ARTIFACTS**

  (Optional) Name of the artifacts folder. Default: "Artifacts"
