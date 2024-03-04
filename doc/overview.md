# Overview

You can set up a Continuous Integration and Continuous Deployment (CI/CD) pipeline for Microsoft Teams apps created with the Teams Toolkit. How you enable the pipeline depends on your needs and scenarios. 

## Teams app CI/CD Starter Guide 
You can use the Teams Toolkit CLI to set up a CI/CD pipeline for your Teams app. This article covers preparing Azure resources and configuring settings, as well as setting up the CI/CD pipeline using GitHub and Azure DevOps platforms. The guide will also walk you through creating and running the pipeline to automate the deployment of code to Azure and generating an appPackage for app distribution.

## Teams app CI/CD transparent guide
You can set up a custom CI/CD pipeline for a Teams app in cases where the Teams Toolkit CLI cannot be used or a custom deployment process is required. This article explains how to create a custom CI/CD pipeline and offers examples of using GitHub official actions and Azure Pipeline tasks for building and deploying projects written in different programming languages and hosted on various Azure services. Additionally, the guide discusses methods of authentication for deploying app code to Azure services using OpenID Connect (OIDC) or a secret, and provides steps to manually create an app package for testing the Teams app if the Teams Toolkit CLI cannot be used to generate the appPackage.zip file.
