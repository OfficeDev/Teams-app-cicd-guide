# Set up CI/CD pipelines

You can set up a Continuous Integration and Continuous Deployment (CI/CD) pipeline for Microsoft Teams apps created with the Teams Toolkit. A Teams app CI/CD pipeline typically consists of three parts: 
1. Building the project
1. Deploying the project to cloud resources
1. Generating the Teams app package.

> Before creating a pipeline for a Teams app, it is essential to prepare the necessary cloud resources, such as Azure Web App, Azure Functions, or Azure Static Web App, and configure the app settings. 

Building the project involves compiling the source code and creating the necessary artifacts for deployment. For the deployment process, it is recommended to use the Teams Toolkit CLI. For more information, please refer to [Set up CI/CD pipelines with Teams Toolkit CLI](#set-up-cicd-pipelines-with-teams-toolkit-cli). If you prefer not to use the Teams Toolkit CLI for deployment or would like to customize your pipeline, you can refer to [Set up CI/CD pipelines using your own workflow](#set-up-cicd-pipelines-using-your-own-workflow). The final step, generating the Teams app package, helps you test your Teams app after deployment by [Upload your app in Teams](https://learn.microsoft.com/en-us/microsoftteams/platform/concepts/deploy-and-publish/apps-upload). 

## Set up CI/CD pipelines with Teams Toolkit CLI
The [Teams Toolkit CLI](https://learn.microsoft.com/en-us/microsoftteams/platform/toolkit/teams-toolkit-cli?pivots=version-three) automates the following procedures:
1.	deploying code to Azure.
2.	generating a Teams app's appPackage, which can be used for [app distribution](https://learn.microsoft.com/en-us/microsoftteams/platform/concepts/deploy-and-publish/apps-publish-overview).

> For creating project, we recommend using Teams Toolkit version 5.6.0 or later.
 
### Prerequisites

> If you already ran Teams Toolkit’s “**provision**” command, you can skip prerequisites 1 and 2.
    
1.	Prepare Teams app needed resources.

    Prepare the resources needed for your app's manifest (teams app id, bot id, etc).  
2.	Prepare and config Azure resources.

    You can manually prepare these resources by looking into biceps files under infra/ folder.

3.	Prepare service principal.

    You should have a service principal and configure its access policies on resources. Below are some docs that you can refer to:
    -	[Create service principal using Azure portal](https://learn.microsoft.com/en-us/entra/identity-platform/howto-create-service-principal-portal)
    -	[Create service principal using Azure CLI](https://learn.microsoft.com/en-us/cli/azure/azure-cli-sp-tutorial-1?tabs=bash#create-a-service-principal-with-role-and-scope)
    
    Teamsapp cli currently supports login to Azure using service principal secret. [Create a secret](https://learn.microsoft.com/en-us/entra/identity-platform/howto-create-service-principal-portal#option-3-create-a-new-client-secret) and save service principal’s **client id**, **client secret**, **tenant id** for following steps.


After you meet the above prerequisites, you can follow the steps below to setup the pipeline:
- [Steps for Github Actions](#steps-for-github-actions)
- [Steps for Azure Pipeline](#steps-for-azure-pipeline)
### Steps for Github Actions

#### 1. Create a CD yml in your project
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
      TEAMSAPP_CLI_VERSION: "3.0.0"
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
>The default pipeline will be triggered when push events happen on master branch, you can modify it to meet your own needs. 
#### 2. Prepare a GitHub repository
See [create a GitHub repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/quickstart-for-repositories).

#### 3. Set variables/secrets in the repository
The following variables and secrets are needed for the pipeline:
- Service principal related: service principal client id, client secret, tenant id. These are needed for login to Azure.
- Deployment related: all placeholders (wrapped in \${{}}) in **teamspp.yml’s deploy** section.
- AppPackage related: all placeholders (wrapped in \${{}}) in **manifest.json**.
> If you used Teams Toolkit's "**provision**" command to create Azure resources and Teams app resources, you can find the values of these variables in your project's /env files.

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

#### 4. Modifications of the yml file
The variables used for **deployment** and **generating appPackage** need to be explicitly added to env so that teamsapp cli can read them.

Taking basic bot template as an example, after modification the yml file will be like:
```yml
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      TEAMSAPP_CLI_VERSION: "3.0.0"
      # Add extra environment variables here so that teamsapp cli can use them.
      BOT_AZURE_APP_SERVICE_RESOURCE_ID: ${{vars.BOT_AZURE_APP_SERVICE_RESOURCE_ID}}
      TEAMS_APP_ID: ${{vars.TEAMS_APP_ID}}
      BOT_ID: ${{vars.BOT_ID}}
      APP_NAME_SUFFIX: ${{vars.APP_NAME_SUFFIX}}

    steps:
    ...
    ...
```

#### 5. Run the pipeline
Push code to the repo to trigger pipeline. 
> You don't need to commit env files under env/ folder into the repo. The env needed for running CI/CD pipeline are already set in the repo variables.

The default pipeline will be triggered when push events happen on master branch, you can modify it to meet your own needs. 

After the pipeline runs successfully you should see from the log that code has been deployed to Azure and the appPackage has been generated in artifacts. You can use this appPackage to [distribute your app](https://learn.microsoft.com/en-us/microsoftteams/platform/concepts/deploy-and-publish/apps-publish-overview).

### Steps for Azure pipeline
#### 1. Create a CD yml in your project
Create a yml file under your project. Write the following content into this yml file.
```yml
trigger:
  - master

pool:
  vmImage: ubuntu-latest

variables:
  TEAMSAPP_CLI_VERSION: 3.0.0

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
>The default pipeline will be triggered when push events happen on master branch, you can modify it to meet your own needs. 
####	2. Prepare Azure or GitHub repository
- [Create Azure repository](https://learn.microsoft.com/en-us/azure/devops/repos/git/create-new-repo?view=azure-devops)
- [Create GitHub repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/quickstart-for-repositories)
#### 3. Setup Azure pipeline
After pushing your code to repo. Go to **Pipelines**, and then select **New pipeline**. Select your repo and select your existing yml file to configure your pipeline.

#### 4. Set variables/secrets in pipeline 
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

#### 5. Run the pipeline
Push code to the repo to trigger pipeline. 
> You don't need to commit env files under env/ folder into the repo. The env needed for running CI/CD pipeline are already set in the pipeline variables.

After the pipeline runs successfully you should see from the log that code deployed to Azure and the appPackage has been generated in artifacts. You can use this appPackage to [distribute your app](https://learn.microsoft.com/en-us/microsoftteams/platform/concepts/deploy-and-publish/apps-publish-overview).


## Set up CI/CD pipelines using your own workflow

The most convenient way to deploy teams app to Azure is using [teamsapp CLI](https://learn.microsoft.com/en-us/microsoftteams/platform/toolkit/teams-toolkit-cli?pivots=version-three)'s `teamspp deploy` command. However, if you're unable to leverage the Teams App CLI within your pipeline, you can create a custom deployment process tailored to your needs.

> If you already have a complete CI/CD pipeline in place for deploying to your Azure resource, your Teams app requires reading environment variables during runtime, you need to configure these environment variables in your Azure resource's settings. You can refer directly to [Prepare app package](#prepare-the-apppackage-for-the-teams-app) section for testing after deployment. 

The `teamsapp deploy` command executes the actions in teamsapp.yml's "deploy" stage. Most of the "deploy" stages consist of "build" and "deploy" actions. To create a custom deployment method, you need to rewrite these actions according to your requirements and preferences.

Taking basic bot typescript project as an example, its teamsapp.yml's deploy stage is as following:
```yml
deploy:
  # Run npm command
  - uses: cli/runNpmCommand
    name: install dependencies
    with:
      args: install
  - uses: cli/runNpmCommand
    name: build app
    with:
      args: run build --if-present
  # Deploy your application to Azure App Service using the zip deploy feature.
  # For additional details, refer to https://aka.ms/zip-deploy-to-app-services.
  - uses: azureAppService/zipDeploy
    with:
      # Deploy base folder
      artifactFolder: .
      # Ignore file location, leave blank will ignore nothing
      ignoreFile: .webappignore
      # The resource id of the cloud resource to be deployed to.
      # This key will be generated by arm/deploy action automatically.
      # You can replace it with your existing Azure Resource id
      # or add it to your environment variable file.
      resourceId: ${{BOT_AZURE_APP_SERVICE_RESOURCE_ID}}
```
These actions do the following things:
- run `npm install` and `npm build` to build the project.
- deploy code to Azure app service.

You can rewrite these actions in your CI/CD pipeline in your own way. 
Below is an example of using GitHub official actions:
```yml
    # build
    - name: Setup Node 18.x
      uses: actions/setup-node@v1
      with:
        node-version: '18.x'
    - name: 'npm install, build'
      run: |
        npm install
        npm run build --if-present

    - name: 'zip artifact for deployment'
      run: |
        zip -r deploy.zip . --include 'node_modules/*' 'lib/*' 'web.config'

    # deploy
    - name: 'Login via Azure CLI'
      uses: azure/login@v1
      with:
        client-id: ${{ vars.CLIENT_ID }}
        tenant-id: ${{ vars.TENANT_ID }}
        subscription-id: ${{ vars.SUBSCRIPTION_ID }}

    - name: 'Run Azure webapp deploy action using azure RBAC'
      uses: azure/webapps-deploy@v2
      with:
        app-name: ${{ vars.AZURE_WEBAPP_NAME }}
        package: deploy.zip
```

Currently, the Teams Toolkit supports Teams app projects written in different programming languages and suitable to be hosted on different Azure services. Below lists some official actions for build and deploy. You can refer to these actions when setting up CI/CD deployment pipelines.

Build:

| language      | GitHub                    |Azure Pipeline
|---------------------------------------------------|-------------------------------|----|
| JS/TS                | [actions/setup-node](https://github.com/actions/setup-node)  |[NodeTool@0](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/node-tool-v0?view=azure-pipelines) | 
| C#      | [actions/setup-dotnet](https://github.com/actions/setup-dotnet)  |[DotNetCoreCLI@2](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/dotnet-core-cli-v2?view=azure-pipelines)|


Deploy:

| resource   | GitHub                  |Azure Pipeline
|---------------------------------------------------|-------------------------------|----|
| Azure App Service               |[azure/webapps-deploy](https://github.com/Azure/webapps-deploy)| [AzureWebApp@1](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/azure-web-app-v1?view=azure-pipelines)
| Azure Functions     |[Azure/functions-action](https://github.com/Azure/functions-action)|[AzureFunctionApp@2](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/azure-function-app-v2?view=azure-pipelines)
|Azure Static Web Apps             |[Azure/static-web-apps-deploy](https://github.com/Azure/static-web-apps-deploy)| [AzureStaticWebApp@0](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/azure-static-web-app-v0?view=azure-pipelines)|

### Credential needed for login to Azure
If you are using CI/CD to deploy app code to Azure app service, Azure functions or Azure container app, you need a service principal to login to Azure. There are 2 ways of login to Azure using service principal: using **OpenID Connect(OIDC)** or **secret**.

- OIDC (recommended)

  OIDC is the recommended way since it has increased security. To use it in GitHub actions, follow this [guide](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-portal%2Cwindows#create-a-microsoft-entra-application-and-service-principal) to create the service principal and add federated credentials, and set the client id , tenant id and sub id in GitHub repo variables, then you can use Azure/login action like following:
  ```yml
  - name: 'Login using OIDC'
        uses: azure/login@v1
        with:
          client-id: ${{ vars.SERVICE_PRINCIPAL_CLIENT_ID }}
          tenant-id: ${{ vars.SERVICE_PRINCIPAL_TENANT_ID }}
          subscription-id: ${{ vars.SERVICE_PRINCIPAL_SUBSCRIPTION_ID }}
  ```
  > To use OIDC in GitHub actions, you need to modify the permission in CI/CD:
  ```yml
  permissions: 
    id-token: write
    contents: read
  ```

  For Azure pipeline, ypu need to create a [workload identity federation service connection](https://learn.microsoft.com/en-us/azure/devops/pipelines/library/connect-to-azure?view=azure-devops#create-an-azure-resource-manager-service-connection-using-workload-identity-federation). Then you can use the connection name in your pipeline:

  ```yml
  - task: AzureWebApp@1
    inputs:
      azureSubscription: $(connection_name)
      appName: $(app_name)
      package: '$(System.DefaultWorkingDirectory)/'
    displayName: 'Deploy to Azure Web App'
  ```

- Secret

  For GitHub actions, follow this [guide](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-portal%2Cwindows#use-the-azure-login-action-with-a-service-principal-secret) to create a service principal and secret. Then you can use Azure/login action like:
  ```yml
      - uses: Azure/login@v1
        with:
          creds: '{"clientId":"${{ secrets.CLIENT_ID }}","clientSecret":"${{ secrets.CLIENT_SECRET }}","subscriptionId":"${{ secrets.SUBSCRIPTION_ID }}","tenantId":"${{ secrets.TENANT_ID }}"}'
  ```
  For Azure pipeline, you need to create a [service principal service connection](https://learn.microsoft.com/en-us/azure/devops/pipelines/library/connect-to-azure?view=azure-devops#create-an-azure-resource-manager-service-connection-using-a-service-principal-secret). Then you can use the service connection name in actions like below:
  ```yml
      - task: AzureWebApp@1
        inputs:
          azureSubscription: $(connection_name)
          appName: $(app_name)
          package: '$(System.DefaultWorkingDirectory)/'
        displayName: 'Deploy to Azure Web App'
  ```
### Prepare the appPackage for the teams app
You will need the `appPackage` to distribute your Teams app. Teamsapp CLI's command "teamsapp package" can help you create the `appPackage.zip` automatically. If you cannot leverage teamsapp CLI to do this, you can follow below steps to create the appPackage by hand.
1. prepare a `appPackage/` folder.

1. put `mainfest.json` in `appPackage` folder.

    The default `manifest.json` in Teams Toolkit project has placeholders (wrapped in ${{}}). You should replace these placeholders with true values.

2. put app icons in `appPackage` folder.
    
    Follow the guide to prepare [app icons](https://learn.microsoft.com/en-us/microsoftteams/platform/concepts/build-and-test/apps-package#app-icons). You should have 2 .png files as output.
3. zip the appPackage folder.
    Zip the `appPackage/` folder into `appPackage.zip`.


