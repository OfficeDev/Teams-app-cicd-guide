# Teams app CI/CD starter guide
This document provides guidance on how to set up a CI/CD pipeline for Teams apps created with the Teams Toolkit, using GitHub and Azure DevOps platform, through the [Teamsapp CLI](https://learn.microsoft.com/en-us/microsoftteams/platform/toolkit/teams-toolkit-cli?pivots=version-three). The pipeline automates the following procedures:
1.	deploying code to Azure.
2.	generating a Teams app's appPackage, which can be used for [app distribution](https://learn.microsoft.com/en-us/microsoftteams/platform/concepts/deploy-and-publish/apps-publish-overview).

> For creating project, we recommend using Teams Toolkit version 5.6.0 or later.

 
### Prerequisites

1.	Prepare Teams app needed resources.

    Prepare the resources needed for your app's manifest (teams app id, bot id, etc). You can leverage Teams Toolkit’s “**provision**” command to create these resources. 
2.	Prepare and config Azure resources.

    You can also leverage Teams Toolkit’s “**provision**” command to create the needed Azure resources, or you can manually prepare these resources by looking into biceps files under infra/ folder.

3.	Prepare service principal.

    You should have a service principal and configure its access policies on resources. Below are some docs that you can refer to:
    -	[Create service prinicapl using Azure portal](https://learn.microsoft.com/en-us/entra/identity-platform/howto-create-service-principal-portal)
    -	[Create service prinicapl using Azure CLI](https://learn.microsoft.com/en-us/cli/azure/azure-cli-sp-tutorial-1?tabs=bash#create-a-service-principal-with-role-and-scope)
    
    Teamsapp cli currently supports login to Azure using service principal secret. [Create a secret](https://learn.microsoft.com/en-us/entra/identity-platform/howto-create-service-principal-portal#option-3-create-a-new-client-secret) and save service principal’s **client id**, **client secret**, **tenant id** for following steps.




After you meet the above prerequites, you can follow the steps below to setup the pipeline:
- [Steps for Github Actions](#steps-for-github-actions)
- [Steps for Azure Pipeline](#steps-for-azure-pipeline)
## Steps for Github Actions

### 1. Create a CD yml in your project
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
          --password ${{secrets.AZURE_SERVICE_PRINCIPAL_CLIENT_SECRET }} \
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
### 2. Prepare a GitHub repository
See [create a GitHub repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/quickstart-for-repositories).

### 3. Set variables/secrets in the repository
The following variables and secrets are needed for the pipeline:
- Service principal related: service principal client id, client secret, tenant id. These are needed for login to Azure.
- Deployment related: all placeholders (wrapped in \${{}}) in **teamspp.yml’s deploy** section.
- AppPackage related: all placeholders (wrapped in \${{}}) in **manifest.json**.

Taking basic bot template as an example, below variables and secrets are needed:
- AZURE_SERVICE_PRINCIPAL_CLIENT_ID, AZURE_TENANT_ID, AZURE_SERVICE_PRINCIPAL_CLIENT_SECRET
- Below is a basic bot template's teamsapp.yml deploy section. It contains a placeholder \${{BOT_AZURE_APP_SERVICE_RESOURCE_ID}}.
```yml
deploy:
  - uses: cli/runNpmCommand
    name: install dependencies
    with:
      args: install
  - uses: cli/runNpmCommand
    name: build app
    with:
      args: run build --if-present
  - uses: azureAppService/zipDeploy
    with:
      artifactFolder: .
      ignoreFile: .webappignore
      resourceId: ${{BOT_AZURE_APP_SERVICE_RESOURCE_ID}} 
```
- Its manifest.json contains these placeholders: \${{TEAMS_APP_ID}}, \${{APP_NAME_SUFFIX}}, \${{BOT_ID}}.

Therefore for basic bot template, you need to set following variables/secrets in the repository.

| name                                              | purpose                       |
|---------------------------------------------------|-------------------------------|
| AZURE_SERVICE_PRINCIPAL_CLIENT_ID                 | Service principal credentials |
| AZURE_TENANT_ID                                   | Service principal credentials |
| AZURE_SERVICE_PRINCIPAL_CLIENT_SECRET             | Service principal credentials |
| BOT_AZURE_APP_SERVICE_RESOURCE_ID | deployment                    |
| BOT_ID                | appPackage                    | 
| TEAMS_APP_ID                 | appPackage                    |
| APP_NAME_SUFFIX               | appPackage                    |

For setting variables/secrets for a GitHub repository, see [creating variables for a repository](https://docs.github.com/en/actions/learn-github-actions/variables#creating-configuration-variables-for-a-repository).
> The  AZURE_SERVICE_PRINCIPAL_CLIENT_SECRET should be set as secret.

### 4. Modifications of the yml file
The variables used for **deployment** and **generating appPackage** need to be explicitly added to env so that teamsapp cli can read them.

Taking basic bot template as an example, after modification the yml file will be like:
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

    steps:
    ...
    ...
```

### 5. Run the pipeline
Push code to the repo to trigger pipeline. 
> You don't need to commit env files under env/ folder into the repo. The env needed for running CI/CD pipeline are already set in the repo variables.

The default pipeline will be triggered when push events happen on master branch, you can modify it to meet your own needs. 

After the pipeline runs successfully you should see from the log that code has been deployed to Azure and the appPackage has been generated in artifacts. You can use this appPackage to [distribute your app](https://learn.microsoft.com/en-us/microsoftteams/platform/concepts/deploy-and-publish/apps-publish-overview).

## Steps for Azure pipeline
### 1. Create a CD yml in your project
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
###	2. Prepare Azure or GitHub repository
- [Create Azure repository](https://learn.microsoft.com/en-us/azure/devops/repos/git/create-new-repo?view=azure-devops)
- [Create GitHub repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/quickstart-for-repositories)
### 3. Setup Azure pipeline
After pushing your code to repo. Go to **Pipelines**, and then select **New pipeline**. Select your repo and select your existing yml file to configure your pipeline.

### 4. Set variables/secrets in pipeline 
The following variables and secrets are needed for the pipeline:
- Service principal related: service principal client id, client secret, tenant id.
- Deployment related: all placeholders (wrapped in \${{}}) in **teamspp.yml’s deploy section**.
- AppPackage related: all placeholders (wrapped in \${{}}) in **manifest.json**.

Taking basic bot template as an example, below variables and secrets are needed:
- AZURE_SERVICE_PRINCIPAL_CLIENT_ID, AZURE_TENANT_ID, AZURE_SERVICE_PRINCIPAL_CLIENT_SECRET
- Below is a basic bot template's teamsapp.yml deploy section. It contains a placeholder \${{BOT_AZURE_APP_SERVICE_RESOURCE_ID}}.
```yml
deploy:
  - uses: cli/runNpmCommand
    name: install dependencies
    with:
      args: install
  - uses: cli/runNpmCommand
    name: build app
    with:
      args: run build --if-present
  - uses: azureAppService/zipDeploy
    with:
      artifactFolder: .
      ignoreFile: .webappignore
      resourceId: ${{BOT_AZURE_APP_SERVICE_RESOURCE_ID}} 
```
- Its manifest.json contains these placeholders: \${{TEAMS_APP_ID}}, \${{APP_NAME_SUFFIX}}, \${{BOT_ID}}.

Therefore, for basic bot template, you need to set the following values and secrets in the pipeline:

| name                                              | purpose                       |
|---------------------------------------------------|-------------------------------|
| AZURE_SERVICE_PRINCIPAL_CLIENT_ID                 | Service principal credentials |
| AZURE_TENANT_ID                                   | Service principal credentials |
| AZURE_SERVICE_PRINCIPAL_CLIENT_SECRET             | Service principal credentials |
| BOT_AZURE_APP_SERVICE_RESOURCE_ID | deployment                    |
| BOT_ID                | appPackage                    | 
| TEAMS_APP_ID                 | appPackage                    |
| APP_NAME_SUFFIX               | appPackage                    |

### 5. Run the pipeline
Push code to the repo to trigger pipeline. 
> You don't need to commit env files under env/ folder into the repo. The env needed for running CI/CD pipeline are already set in the pipeline variables.

The default pipeline will be triggered when push events happen on master branch, you can modify it to meet your own needs. 

After the pipeline runs successfully you should see from the log that code deployed to Azure and the appPackage has been generated in artifacts. You can use this appPackage to [distribute your app](https://learn.microsoft.com/en-us/microsoftteams/platform/concepts/deploy-and-publish/apps-publish-overview).
