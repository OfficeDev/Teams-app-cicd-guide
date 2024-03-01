# Teams app CI/CD starter guide
This document provides guidance on how to set up a CI/CD pipeline for Teams apps created with the Teams Toolkit, using GitHub and Azure DevOps platform, through the [Teamsapp CLI](https://learn.microsoft.com/en-us/microsoftteams/platform/toolkit/teams-toolkit-cli?pivots=version-three). The pipeline automates the following procedures:
1.	deploying code to Azure.
2.	generating a Teams app's appPackage, which can be used for [app distribution](https://learn.microsoft.com/en-us/microsoftteams/platform/concepts/deploy-and-publish/apps-publish-overview).

You should follow the steps below to setup the pipeline: 
### Prerequisites
1.	Prepare and config Azure resources.

    You can leverage Teams Toolkit’s “**provision**” command to create the needed Azure resources, or you can manually prepare these resources by looking into biceps files under infra/ folder.

2.	Prepare service principal.

    You should have a service principal and configure its access policies on resources. Below are some docs that you can refer to:
    -	Azure portal: https://learn.microsoft.com/en-us/entra/identity-platform/howto-create-service-principal-portal
    -	Azure CLI: https://learn.microsoft.com/en-us/cli/azure/azure-cli-sp-tutorial-1?tabs=bash#create-a-service-principal-with-role-and-scope

    Save service principal’s **client id**, **client secret**, **tenant id** for following steps.

3.	Prepare Teams app needed resources.

    Prepare the resources needed for your app's manifest( teams app id, bot id, etc). You can leverage Teams Toolkit’s “**provision**” command to create these ids. 
## Steps (GitHub)

### Prepare a GitHub repository

### Set variables/secrets in the repository
The following variables and secrets are needed for the pipeline:
1. Service principal related: service principal client id, client secret, tenant id.
2. Deployment related: all placeholders (wrapped in \${{}}) in teamspp.yml’s deploy section.
3. AppPackage related: all placeholders (wrapped in \${{}}) in manifest.json.

    Taking basic bot template as an example, the teamsapp.yml’s deploy section contains a placeholder \${{BOT_AZURE_APP_SERVICE_RESOURCE_ID}} indicating the resource id of the app service that the code will be deployed to. The manifest.json contains the following placeholders: \${{TEAMS_APP_ID}}, \${{APP_NAME_SUFFIX}}, \${{BOT_ID}}. These vars need to be set in the repository.
Therefore, for basic bot template, you need to set the following values and secrets:

| name                                              | purpose                       |
|---------------------------------------------------|-------------------------------|
| AZURE_SERVICE_PRINCIPAL_CLIENT_ID                 | Service principal credentials |
| AZURE_TENANT_ID                                   | Service principal credentials |
| AZURE_SERVICE_PRINCIPAL_CLIENT_SECRET             | Service principal credentials |
| BOT_AZURE_APP_SERVICE_RESOURCE_ID | deployment                    |
| BOT_ID                | appPackage                    | 
| TEAMS_APP_ID                 | appPackage                    |
| APP_NAME_SUFFIX               | appPackage                    |

> The  AZURE_SERVICE_PRINCIPAL_CLIENT_SECRET should be set as secret.
### Create a CD yml
Create a cd.yml file under .github/workflows/ folder.  Write the following content into this yml file.
```yml
on:
  push:
    branches:
      - master
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      TEAMSAPP_CLI_VERSION: "3.0.0-beta.2024012307.0"
      # Add extra environment variables here so that teamsapp cli can use them.

    steps:
      - name: "Checkout Github Action"
        uses: actions/checkout@master

      - name: Setup Node 18.x
        uses: actions/setup-node@v1
        with:
          node-version: "18.x"

      - name: install cli
        run: |
          npm install @microsoft/teamsapp-cli@${{env.TEAMSAPP_CLI_VERSION}}

      - name: Login Azure by service principal
        run: |
          npx teamsapp account login azure --username ${{vars.AZURE_SERVICE_PRINCIPAL_CLIENT_ID}}  \
          --service-principal true \
          --tenant ${{vars.AZURE_TENANT_ID}} \
          --password ${{secrets.AZURE_SERVICE_PRINCIPAL_ CLIENT_SECRET }} \
          --interactive false

      - name: Deploy to hosting environment
        run: |
          npx teamsapp deploy --ignore-env-file true \
          --interactive false

      - name: Package app
        run: |
          npx teamsapp package

      - name: upload appPackage
        uses: actions/upload-artifact@v4
        with:
          name: artifact
          path: appPackage/build/appPackage.zip


```
### Additional modifications of the yml file
The variables used for deployment and generating appPackage will need to be explicitly added to env so that teamsapp cli can read them.

Taking basic bot template as an example, you need to manually set the following values into env in yml:
```yml
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      TEAMSAPP_CLI_VERSION: "3.0.0-beta.2024012307.0"
      # Add extra environment variables here so that teamsapp cli can use them.
      BOT_AZURE_APP_SERVICE_RESOURCE_ID: ${{vars.BOT_AZURE_APP_SERVICE_RESOURCE_ID}}
      TEAMS_APP_ID: ${{vars.TEAMS_APP_ID}}
      BOT_ID: ${{vars.BOT_ID}}
      APP_NAME_SUFFIX: ${{vars.APP_NAME_SUFFIX}}
```

### Run the pipeline
Push code to the repo to trigger pipeline. 

The default pipeline will be triggered when push events happen on master branch, you can modify it to meet your own needs. 

After the pipeline runs successfully you should see from the log that code deployed to Azure and the appPackage has been generated in artifacts. You can use this appPackage to [distribute your app](https://learn.microsoft.com/en-us/microsoftteams/platform/concepts/deploy-and-publish/apps-publish-overview).

## Steps (Azure pipeline)
###	Prepare Azure or GitHub repository
### Create a CD yml
Create a yml file under your project. Write the following content into this yml file.
```yml
trigger:
  - master

pool:
  vmImage: ubuntu-latest

variables:
  TEAMSAPP_CLI_VERSION: 3.0.0-beta.2024012307.0

steps:
  - task: NodeTool@0
    inputs:
      versionSpec: "18"
      checkLatest: true

  - script: |
      npm install @microsoft/teamsapp-cli@$(TEAMSAPP_CLI_VERSION)
    displayName: "Install CLI"

  - script: |
      npx teamsapp account login azure --username $(AZURE_SERVICE_PRINCIPAL_CLIENT_ID) --service-principal true --tenant $(AZURE_TENANT_ID) --password $(AZURE_SERVICE_PRINCIPAL_CLIENT_SECRET) --interactive false
    displayName: "Login Azure by service principal"

  - script: |
      npx teamsapp deploy --ignore-env-file true --interactive false
    displayName: "Deploy to Azure"

  - script: |
      npx teamsapp package
    displayName: "Package app"

  - publish: $(System.DefaultWorkingDirectory)/appPackage/build/appPackage.zip
    artifact: artifact
```
### Setup Azure pipeline
After pushing your code to repo. Go to **Pipelines**, and then select **New pipeline**. Select your repo and select your existing yml file to configure your pipeline.

### Set variables/secrets in repository 
The following variables and secrets are needed for the pipeline:
1. Service principal related: service principal client id, client secret, tenant id.
2. Deployment related: all placeholders (wrapped in \${{}}) in teamspp.yml’s deploy section.
3. AppPackage related: all placeholders (wrapped in \${{}}) in manifest.json.

    Taking basic bot template as an example, the teamsapp.yml’s deploy section contains a placeholder \${{BOT_AZURE_APP_SERVICE_RESOURCE_ID}} indicating the resource id of the app service that the code will be deployed to. The manifest.json contains the following placeholders: \${{TEAMS_APP_ID}}, \${{APP_NAME_SUFFIX}}, \${{BOT_ID}}. These need to be set in the repository.
Therefore, for basic bot template, you need to set the following values and secrets:

| name                                              | purpose                       |
|---------------------------------------------------|-------------------------------|
| AZURE_SERVICE_PRINCIPAL_CLIENT_ID                 | Service principal credentials |
| AZURE_TENANT_ID                                   | Service principal credentials |
| AZURE_SERVICE_PRINCIPAL_CLIENT_SECRET             | Service principal credentials |
| BOT_AZURE_APP_SERVICE_RESOURCE_ID | deployment                    |
| BOT_ID                | appPackage                    | 
| TEAMS_APP_ID                 | appPackage                    |
| APP_NAME_SUFFIX               | appPackage                    |

### Run the pipeline
Push code to the repo to trigger pipeline. 

The default pipeline will be triggered when push events happen on master branch, you can modify it to meet your own needs. 

After the pipeline runs successfully you should see from the log that code deployed to Azure and the appPackage has been generated in artifacts. You can use this appPackage to [distribute your app](https://learn.microsoft.com/en-us/microsoftteams/platform/concepts/deploy-and-publish/apps-publish-overview).
