# Overview

You can set up a Continuous Integration and Continuous Deployment (CI/CD) pipeline for Microsoft Teams apps created with the Teams Toolkit. There are few prerequisites that you must go through before working on this.

Before creating a pipeline for a Teams app, it is essential to prepare the necessary cloud resources, such as Azure Web App, Azure Functions, or Azure Static Web App, and configure the app settings. A Teams app CI/CD pipeline typically consists of three parts: building the project, deploying the project to cloud resources, and building the Teams app package.

Building the project involves compiling the source code and creating the necessary artifacts for deployment. For the deployment process, it is recommended to use the Teams App CLI. For more information, please refer to [Teams app CI/CD pipeline starter guide](/doc/getstarted.md). If you prefer not to use the Teams App CLI for deployment or would like to customize your pipeline, you can refer to [Alternative Deployment Solutions for Teams App CI/CD Pipelines](/doc/alternativesolution.md).

The final step, building the Teams app package, helps you test your Teams app after deployment by [Upload your app in Teams](https://learn.microsoft.com/en-us/microsoftteams/platform/concepts/deploy-and-publish/apps-upload). By following these guidelines and ensuring that the necessary cloud resources and configurations are in place, you can streamline the process of creating and deploying a Teams app using a CI/CD pipeline.
