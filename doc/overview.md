# Overview

You can set up a Continuous Integration and Continuous Deployment (CI/CD) pipeline for Microsoft Teams apps created with the Teams Toolkit. How you enable the pipeline depends on your needs and scenarios. 

## Teams app CI/CD Starter Guide 
The CI/CD starter guide is for those who can use the Teams Toolkit CLI, focuses on quickly setting up a CI/CD pipeline for your Teams app. It covers the essentials, such as preparing Azure resources, configuring settings, and utilizing GitHub and Azure DevOps platforms. By following this guide, you will learn how to create and run the pipeline, automating the deployment of code to Azure and generating an appPackage for app distribution.

## Teams app CI/CD transparent guide
If you cannot use the Teams Toolkit CLI or require a custom deployment process for your Teams app, the CI/CD transparent guide provides a detailed guide on creating a custom CI/CD pipeline. This guide shows the use of GitHub official actions and Azure Pipeline tasks for building and deploying teams app project written in different programming languages and hosted on various Azure services. It also covers methods of authentication for deploying app code to Azure services using OpenID Connect (OIDC) or a secret. Moreover, it offers steps to manually create an app package for testing the Teams app if the Teams Toolkit CLI cannot be used to generate the appPackage.zip file.
