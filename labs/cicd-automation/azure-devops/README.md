# Lab: Azure DevOps CI/CD

This workshop will guide you through building Continuous Integration (CI) and Continuous Deployment (CD) pipelines with Azure DevOps for use with Azure Kubernetes Service. The pipeline will utilize Azure Container Registry to build the images and Helm for application updating. 

## Prerequisites 

* Complete previous labs:
    * [Azure Kubernetes Service](../../create-aks-cluster/README.md)
    * [Build Application Components in Azure Container Registry](../../build-application/README.md)
    * [Helm Setup and Deploy Application](../../helm-setup-deploy/README.md)

## Instructions

The general workflow/result will be as follows:

* Push code to source control
* Trigger a continuous integration (CI) build pipeline when project code is updated via Git
* Package app code into a container image (Docker Image) created and stored with Azure Container Registry
* Trigger a continuous deployment (CD) release pipeline upon a successful build
* Deploy container image to AKS upon successful a release (via Helm chart)
* Rinse and repeat upon each code update via Git
* Profit

![Azure DevOps Workflow](workflow.png)

#### Setup Azure DevOps Project

1. Create a Azure DevOps organization/account. Follow the steps here: https://docs.microsoft.com/en-us/azure/devops/user-guide/sign-up-invite-teammates?view=vsts

2. Create New Project in Azure DevOps

    * Name your project "kubernetes-hackfest" and give it a description.
    * Leave the Version control as Git

    ![Azure DevOps New Project](azure-do-new-project.png)

3. On the next screen, click "Repos" and then "import a repository"

    Enter `https://github.com/Azure/kubernetes-hackfest` for the Clone URL and click "Import"
    
    ![Azure DevOps Import](azure-do-import.png)

#### Create Build Pipeline

1. In Azure DevOps, click on "Pipelines" on the left menu and then click "Builds"

2. Click the "New pipeline" button

3. In "Select a source," use `Azure Repos Git` and ensure it is pointing to your newly built repo (this is the default)
    > Note that we are using the master branch here. Normally we would use other branches and PR's. For simplicity, we are using master just for this lab.

4. Select to "start with an Empty job"

5. Leave the name as "kubernetes-hackfest-CI"

6. Change the Agent pool to use the "Hosted Ubuntu 1604"

7. Click the plus sign by "Agent job 1" to add a task

8. Search tasks for "Azure" and add the Azure CLI task

    ![Azure DevOps Azure CLI](azure-do-azurecli.png)

9. Click on the Azure CLI task and choose your Azure subscription and `Authorize`

10. For "Script Location" choose "Inline script" and enter the following (be sure to replace the ACR name with yours). 

    > Note: We are creating a dynamic image tag using our build ID from Azure DevOps.

    ```bash
    # set your Azure Container Registry name below
    export ACRNAME=
    export IMAGETAG=azuredevops-$(Build.BuildId)

    az acr build -t hackfest/data-api:$IMAGETAG -r $ACRNAME --no-logs ./app/data-api
    az acr build -t hackfest/flights-api:$IMAGETAG -r $ACRNAME --no-logs ./app/flights-api
    az acr build -t hackfest/quakes-api:$IMAGETAG -r $ACRNAME --no-logs ./app/quakes-api
    az acr build -t hackfest/weather-api:$IMAGETAG -r $ACRNAME --no-logs ./app/weather-api
    az acr build -t hackfest/service-tracker-ui:$IMAGETAG -r $ACRNAME --no-logs ./app/service-tracker-ui  
    ```

    ![Azure DevOps CLI](azure-do-cli.png)

11. Add another task to "Agent job 1" and search for "Publish Build Artifacts". Use "charts" for the artifact name and browse to the charts folder for the "Path to publish"

    ![Azure DevOps Artifact](azure-do-artifact.png)

12. Test this by clicking "Save & queue" and providing a comment

13. Click on "Builds" to check result. It can take a bit of time for all of the steps to complete. 

14. Enable Continuous integration for the build definition. Edit the build definition and you will find this setting under "Triggers"

#### Create Deployment Pipeline

In the deployment pipeline, we will create a Helm task to update our application. 

    > Note: To save time, we will only deploy the service-tracker-ui application in this lab. 

1. In Azure DevOps, click on "Pipelines" on the left menu and then click "Releases"

2. Click the "New pipeline" button

3. Select to "start with an Empty job"

4. Name the pipeline "AKS Helm Deploy" (it will default to "New release pipeline")

5. Click on "+ Add" next to Artifacts

6. In "Source (build pipeline)", select the build we created earlier (should be named "kubernetes-hackfest-CI")

    ![Azure DevOps Release Artifact](azure-do-release-artifact.png)

7. Click on the lightning bolt next to the Artifact we just created and enable "Continuous deployment trigger"

8. Click on "Stage 1" in the Stages box.

9. Name the stage "dev"

10. Click on "1 job, 0 task" to view stage tasks

11. Click on "Agent job" and change the Agent pool to "Hosted Ubuntu 1604" in the drop down

12. On the Agent job, click the "+" to add a Task

13. Search for "helm" and add the task called "Helm Tool Installer" as first task. Click Add

14. Next, Search for "helm" and add the task called "Package and deploy Helm charts". Click Add

15. Click on the task (named "helm ls") to configure all of the settings for the release
    
    * Select your Azure subscription in the dropdown and click "Authorize"
    * Select the Resource Group and AKS Cluster
    * For Namespace, enter "hackfest"
    * For the Command select "upgrade"
    * For Chart type select "File Path"
    * For Chart path, click the "..." button and browse to the "service-trakcer-ui" chart in the charts directory
    * For the Release Name, enter `service-tracker-ui`
    * For Set Values you will need to fix the ACR server to match your ACR server name and the imageTag needs to be set.
        Eg - `deploy.acrServer=acrhackfestbrian13932.azurecr.io,deploy.imageTag=azuredevops-$(Build.BuildId)`

    ![Azure DevOps Helm Task](azure-do-helm-task.png)

    * Click "Save"

#### Run a test build

1. In Azure DevOps, click on Builds and click the "Queue" button

2. Monitor the builds and wait for the build to complete

    ![Azure DevOps Build](azure-do-build.png)

3. The release will automatically start when the build is complete (be patient, this can take some time). Review the results as it is complete. 

    ![Azure DevOps Release](azure-do-release.png)

4. Validate that your newly built image was deployed in your AKS cluster. Eg - `kubectl describe pod...`

5. Now kick-off the full CI/CD pipeline by making an edit to the service-tracker-ui frontend code in the Azure DevOps code repo.

## Troubleshooting / Debugging

* We've seen issues where people forgot to use the hackfest namespace in Kubernetes. 
* Ensure that are using the "Hosted Ubuntu 1604" image in Azure DevOps. 

## Docs / References

* Blog post by Jessica Dean. http://jessicadeen.com/tech/microsoft/how-to-deploy-to-kubernetes-using-helm-and-vsts 

#### Next Lab: [Networking](../../networking/README.md)
