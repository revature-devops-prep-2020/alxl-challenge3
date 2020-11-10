# Challenge 3
Build an Azure CI/CD pipeline for a Dockerized Spring/Node/DotNet microservice on a Kubernetes environment. Set up your cloud infrastructure using the Azure CLI, register a GitHub hook to trigger a build, and validate code quality with SonarCloud before publishing a Docker image to DockerHub. Use `kubectl` to deploy the image to both a testing and staging environment. Report build, test, and deployment results via Slack/Discord or email message.

Include a custom ruleset for SonarCloud, a quality gate to fail a build under these rules, and integrate AquaSecurity's Trivy into the pipeline to scan for docker image vulnerabilities.

Only use Azure services: AKS, ACR, Azure DevOps Pipeline, Azure Repo

## Configure Target Repo
This pipeline expects a Gradle project with a Dockerfile already created. The target project should also have all necessary Kubernetes resource YAMLs in a `kube/` directory. Additional necessary files are located within this repo's `target/` directory; simply place the contents of `target/` in the root of the target project's repo.

The pipeline uses an ACR to store images of the built applications. As such, be sure to change the Kubernetes resource YAMLs to point to the correct images.
```yaml
# using an ACR called MyACR
image: myacr.azurecr.io/image-name:tag
```
The URL to use will also be obtained when the ACR is created.

## Install Azure CLI
Follow the [official documentation](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) to install the Azure CLI for your system.
Once installed, add the DevOps extension:
```sh
az extension add --name azure-devops
```
Log in to begin working:
```sh
az login
```

## Create Azure Resources
### Resource Group
First, make a resource group using the following command:
```sh
# Create the Resource Group
az group create \
    --location eastus \
    --name MyResourceGroup

# Set as the default group for subsequent commands
az configure -d group=MyResourceGroup
```
Note: if you want to see available location names, use this command:
```sh
az account list-locations --query "sort_by([].{Location:name}, &Location)" -o table
```

### Create ACR
If the ACR is made first, it can easily be attached to an AKS cluster when making the cluster. There are [many optional arguments](https://docs.microsoft.com/en-us/cli/azure/acr?view=azure-cli-latest#az_acr_create) to suit your exact needs or preferences. The different SKU options can be compared [here](https://azure.microsoft.com/en-us/pricing/details/container-registry/). For this project, I went with the following:
```sh
az acr create \
    --name MyACR \
    --sku Basic
```

The output should contain the login URL like this:
```json
{
    ...
    "loginServer": "myacr.azurecr.io",
    ...
}
```
That URL should be used for the images mentioned in your Kubernetes resources (see [Configure Target Repo](#configure-target-repo))

### Create AKS Cluster
As with the ACR, there are [many optional arguments](https://docs.microsoft.com/en-us/cli/azure/aks?view=azure-cli-latest#az_aks_create) for creating an AKS cluster. For this project I went with the following:
```sh
az aks create \
    -n MyCluster \
    --attach-acr MyACR \
    --node-count 1 \
    --generate-ssh-keys
```
Once it is finished (which can take several minutes), use the following command to add the context to your kubeconfig so you can access the cluster with `kubectl`:
```sh
az aks get-credentials -n MyCluster
```

If you want to attach/detach an ACR to an already-existing AKS cluster, use the following commands:
```sh
# Attach
az aks update -n MyCluster --attach-acr MyACR

# Detach
az aks update -n MyCluster --detach-acr MyACR
```

### Create Cluster Namespaces
The pipeline expects the Kubernetes namespaces `revcog-test` and `revcog-prod` for test and production environments respectively.
```sh
# Apply the namespaces from the YAML in this repo
kubectl apply -f kube/

# Alternatively, create the namespaces manually
kubectl create ns revcog-test
kubectl create ns revcog-prod
```

## Create Azure DevOps Project
Currently, creating new Azure DevOps Organizations through the CLI is not supported; this is instead done on the [Azure DevOps website](https://dev.azure.com/). Once an organization has been created, you can set it as the default organization for subsequent commands:
```sh
az devops configure -d organization=https://dev.azure.com/MyOrgName
```
Then, you can create a new project for the organization:
```sh
# Make new project
az devops project create --name MyProjectName

# Set it as the default project for subsequent commands
az devops configure -d project=MyProjectName
```

## Commit Code to the Azure DevOps Repo
Use the following command to find the remote URL for your repo:
```sh
az repos list
```
The output should contain the git remote URL like this:
```json
[
    {
        ...
        "remoteUrl": "https://MyOrg@dev.azure.com/MyOrg/MyProject/_git/MyOrg",
        ...
    }
]
```
Using that remote URL, you can now use git to add your project to the Azure DevOps Repo:
```sh
git remote add origin https://MyOrg@dev.azure.com/MyOrg/MyProject/_git/MyOrg
git push -u origin --all
```

## Configure Azure DevOps
Add the SonarCloud pipeline plugin to your organization [here](https://marketplace.visualstudio.com/items?itemName=SonarSource.sonarcloud).

Although it is possible to create service connections with the CLI, it is a very involved process. It is far simpler to navigate to the Service Connections section of the Project Settings on the [Azure DevOps website](https://dev.azure.com/). For this pipeline, the following connections are required:
* a Docker Registry connection called `docker-cr-conn`
* a Kubernetes connection called `kube-conn`
* a SonarCloud connection called `sonar-conn`
Because all of these connections are made to other Azure services, configuring them is extremely easy.

Steps to configure Slack integration can be found [here](https://docs.microsoft.com/en-us/azure/devops/pipelines/integrations/slack?view=azure-devops).

## Create the Pipeline
Use the following command to create the pipeline.
```sh
az pipelines create \
    --name 'MyPipeline' \
    --description 'Pipeline for MyRepo' \
    --repository MyRepoName \
    --repository-type tfsgit \
    --branch main \
    --yml-path azure-pipelines.yml
```
The pipeline will automatically run upon creation unless you specify the flag `--skip-first-run true`.

## After a Run
Once the pipeline has finished running, you can navigate to the results on the Azure DevOps website. The `Tests` tab gives results on both the JUnit tests of your project as well as the results of Trivy's vulnerability scans. The `Extensions` tab provides information from the SonarCloud report.

To find your deployed application, use the following command:
```sh
kubectl get svc -n revcog-prod
```
This will return an external IP; navigate to that IP and the correct port to see your application live.