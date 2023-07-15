# DataFactory-CI-CD
 Explore this public GitHub repository for a detailed guide on setting up CI/CD for Azure Data Factory. Learn best practices, configuration steps, and deployment pipelines to seamlessly integrate, test, and deploy your data engineering workflows. Boost efficiency in your Data Factory projects with efficient CI/CD practices.

## The Importance of CI/CD for Data Factory

- **Improved User Access Control**: CI/CD pipelines offer granular control over user permissions within Data Factory. Access to critical resources, such as production data lakes and databases, can be restricted, reducing the risk of unauthorized data manipulation or exposure.
- **Reduced Security Risks**: With CI/CD pipelines, users do not require direct access to production data lakes, databases, and other sensitive resources. This minimizes the attack surface and mitigates potential security risks associated with unauthorized access or accidental data modification.
- **Code Review and Approval**: CI/CD promotes collaboration and ensures changes adhere to best practices through structured code review and approval processes.
- **Versioning and Rollbacks**: CI/CD enables tracking and easy rollbacks to previous working states, minimizing the impact of errors or unintended consequences.
- **Automated Testing**: CI/CD integrates automated testing processes to validate pipeline configurations and maintain data quality and reliability.

By implementing CI/CD practices, Data Factory can enhance security, maintain data integrity, and streamline the development process.


## Streamlined CI/CD Approach: Automating Software Delivery
![Image](./DataFactory-Devops.jpg)

## Before commencing this tutorial, it is important to establish the following assumptions:

- Two ADF Workspaces, namely Dev and Prod, have already been deployed.
- Two Data Lakes, Dev and Prod, are available.
- Please note that in this scenario, we are considering two ADF instances. However, in real-world scenarios, a testing ADF may also be present.
- The Dev ADF is configured with a linked service to the Dev Data Lake, while the Prod ADF is configured with a linked service to the Prod Data Lake.

## Prerequisites

Before using this pipeline, ensure that you have the following:

- Azure subscription with appropriate access and permissions.
- Azure DevOps account with a connected GitHub repository.
- Azure Pipelines configured and connected to your GitHub repository.

These assumptions will serve as a foundation for the tutorial, ensuring that the subsequent steps and explanations align with the specified environment.

``` It is important to note that only the Dev ADF will be integrated with Git, while the remaining ADF instances such as Prod and Test will not be GIT integrated.```

##Adding the package.json
=======================

Before you start creating the pipeline, you will have to create a `package.json` file. This file will contain the details to obtain the ADFUtilities package. The content of the file is given below:

In the repository, we will create a `build` folder (Folder name can be anything).
Inside the folder, create a `package.json` file.
Paste the following code into the `package.json` file:

```json
{
    "scripts":{
        "build":"node node_modules/@microsoft/azure-data-factory-utilities/lib/index",
        "build-preview":"node node_modules/@microsoft/azure-data-factory-utilities/lib/index --preview"
    },
    "dependencies":{
        "@microsoft/azure-data-factory-utilities":"^1.0.0"
    }
}
```
![Image](./images/adf-npm.gif)

## Pipeline Overview
![Image](./images/PIPELINE.gif)
The pipeline consists of the following stages:

### Stage 1: Build_Adf_Stage

This stage builds the ADF ARM templates and exports them as artifacts. It includes the following tasks:

- **NodeTool**: Installs Node.js to execute the build process.
- **Npm**: Installs npm packages required for building and validating the ADF ARM templates.
- **Validate**: Executes a custom NPM script to validate the ARM templates and perform additional checks.
- **Validate and Generate ARM template**: Executes a custom NPM script to generate the ARM templates and export them.
  ```yml
   - stage: Build_Adf_Arm_Stage
   jobs:
   - job: Build
     pool:
       name: adfcd
       image: ubuntu
     steps:
     - task : NodeTool@0
       displayName: 'Install Node.js'
       inputs:
         versionSpec: '14.x'

     - task : Npm@1
       displayName: 'Install npm package'
       inputs:
         command: 'install'
         workingDir: '$(Build.Repository.LocalPath)/build' 
         verbose: true

     - task: Npm@1
       displayName: 'Validate'
       inputs:
        command: 'custom'
        workingDir: '$(Build.Repository.LocalPath)/build' 
        customCommand: 'run build validate $(Build.Repository.LocalPath)/ $(adfResourceId)'
  
 
     - task: Npm@1
       displayName: 'Validate and Generate ARM template'
       inputs:
         command: 'custom'
         workingDir: '$(Build.Repository.LocalPath)/build' 
         customCommand: 'run build export $(Build.Repository.LocalPath)/ $(adfResourceId) "armTemplate"'

     - task: PublishPipelineArtifact@1
       inputs:
         targetPath: '$(Build.Repository.LocalPath)/build/armTemplate'
         artifact: '$(adfName)-armTemplate'
         publishLocation: 'pipeline'


### Stage 2: Deploy_Adf_DEV_live_mode

This stage deploys the ADF ARM templates to the development environment. It includes the following tasks:

- **Bash**: Installs PowerShell to execute PowerShell scripts.
- **DownloadPipelineArtifact**: Downloads the build artifacts (ADF ARM templates) from the previous stage.
- **AzurePowerShell**: Executes a PowerShell script to perform pre-deployment operations, such as initializing resources and executing custom logic.
- **AzureResourceManagerTemplateDeployment**: Deploys the ADF ARM templates to the development environment using Azure Resource Manager.
- **AzurePowerShell**: Executes a PowerShell script to perform post-deployment operations, such as cleanup or finalization tasks.
``` yml
 - stage: Deploy_Adf_Arm_Stage
   jobs:
   - job: Deploy_to_Live
     pool:
      name: adfcd
       
     steps:
     - task: Bash@3
       inputs:
         targetType: 'inline'
         script: |
           sudo apt-get install -y powershell
           pwsh -Command "Install-Module -Name Az -Force"
     - task : DownloadPipelineArtifact@2
       displayName: Download Build Artifacts - ADF ARM templates
       inputs:
         artifactName: '$(adfName)-armTemplate'
         targetPath: '$(Pipeline.Workspace)/$(adfName)-armTemplate'
     - task: AzurePowerShell@5
       inputs:
         azureSubscription: 'DEV-TEST-SP'
         ScriptType: 'FilePath'
         ScriptPath: '$(Pipeline.Workspace)/$(adfName)-armTemplate/PrePostDeploymentScript.ps1'
         ScriptArguments: '-armTemplate "$(Pipeline.Workspace)/$(adfName)-armTemplate/ARMTemplateForFactory.json" -ResourceGroupName $(resourceGroupName) -DataFactoryName $(adfName) -predeployment $true -deleteDeployment $false'
         azurePowerShellVersion: 'LatestVersion'
         pwsh: true

     - task: AzureResourceManagerTemplateDeployment@3
       inputs:
         deploymentScope: 'Resource Group'
         azureResourceManagerConnection: 'DEV-TEST-SP'
         subscriptionId: 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
         action: 'Create Or Update Resource Group'
         resourceGroupName: '$(resourceGroupName)'
         location: 'North Europe'
         templateLocation: 'Linked artifact'
         csmFile: '$(Pipeline.Workspace)/$(adfName)-armTemplate/ARMTemplateForFactory.json'
         csmParametersFile: '$(Pipeline.Workspace)/$(adfName)-armTemplate/ARMTemplateParametersForFactory.json'
         deploymentMode: 'Incremental'

     - task: AzurePowerShell@5
       inputs:
         azureSubscription: 'DEV-TEST-SP'
         ScriptType: 'FilePath'
         ScriptPath: '$(Pipeline.Workspace)/$(adfName)-armTemplate/PrePostDeploymentScript.ps1'
         ScriptArguments: '-armTemplate "$(Pipeline.Workspace)/$(adfName)-armTemplate/ARMTemplateForFactory.json" -ResourceGroupName $(resourceGroupName) -DataFactoryName $(adfName) -predeployment $false -deleteDeployment $true'
         azurePowerShellVersion: 'LatestVersion'
```
### Stage 3: Deploy_Adf_prod

This stage deploys the ADF ARM templates to the production environment. It follows a similar structure to the Deploy_Adf_Arm_Stage but includes additional tasks for deploying to the production environment.

- **Bash**: Installs PowerShell to execute PowerShell scripts.
- **DownloadPipelineArtifact**: Downloads the build artifacts (ADF ARM templates) from the previous stage.
- **AzurePowerShell**: Executes a PowerShell script to perform pre-deployment operations specific to the production environment.
- **AzureResourceManagerTemplateDeployment**: Deploys the ADF ARM templates to the production environment using Azure Resource Manager.
- overrideParameters field: This field is used to provide custom parameter values during the deployment. In the example, the parameter values being overridden are factoryName, AzureDataLakeStorage1_properties_typeProperties_url, and AzureDataLakeStorage1_accountKey.
- **AzurePowerShell**: Executes a PowerShell script to perform post-deployment operations specific to the production environment.
![Image](./images/adf-pipeline.png)
``` This repository serves as a sample demonstration of implementing CI/CD practices. As a result, for the sake of simplicity, storage account secrets are directly stored as variables in this repository. However, in real-life scenarios, it is recommended to securely store sensitive information like passwords and access keys in Azure Key Vault or other secure vault solutions.```

##[Click here to download the complete script](ADF/azure-pipelines.yml)

For any further assistance or inquiries, please feel free to reach out at snhaider9977@gmail.com
