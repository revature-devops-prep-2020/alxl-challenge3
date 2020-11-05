# Challenge 3
Build an Azure CI/CD pipeline for a Dockerized Spring/Node/DotNet microservice on a Kubernetes environment. Set up your cloud infrastructure using the Azure CLI, register a GitHub hook to trigger a build, and validate code quality with SonarCloud before publishing a Docker image to DockerHub. Use `kubectl` to deploy the image to both a testing and staging environment. Report build, test, and deployment results via Slack/Discord or email message.

Include a custom ruleset for SonarCloud, a quality gate to fail a build under these rules, and integrate AquaSecurity's Trivy into the pipeline to scan for docker image vulnerabilities.

## Configure Target Repo
This pipeline expects a Gradle application with a Dockerfile already created. Additional necessary files are located within this repo's `target/` directory; simply place the contents of `target/` in the root of the target repo.

## Install Azure CLI
[Official documentation](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) to install Azure CLI.
Once installed, add the devops extension:
```sh
az extension add --name azure-devops
```
Log in to begin working:
```sh
az login
```

## Create AKS Cluster
First, make a resource group using the following command:
```sh
# Create the Resource Group
az group create \
    --location eastus
    --name MyResourceGroup

# Set as the default group for subsequent commands
az configure -d group=MyResourceGroup
```
Note: if you want to see available location names, use this command:
```sh
az account list-locations --query "sort_by([].{Location:name}, &Location)" -o table
```
Now, create the cluster; there are [many optional arguments](https://docs.microsoft.com/en-us/cli/azure/aks?view=azure-cli-latest#az_aks_create) for this to meet your exact needs, but for this project I went with the following:
```sh
az aks create \
    -n MyCluster \
    --node-count 1 \
    --generate-ssh-keys
```
Once it is finished (which can take severl minutes), use the following command to add the context to your kubeconfig so you can access the cluster with `kubectl`:
```sh
az aks get-credentials -n MyCluster
```
The pipeline expects the Kubernetes namespaces `revcog-test` and `revcog-prod` for test and production environments respectively.
```sh
# Apply the namespaces from the YAML in this repo
kubectl apply -f kube/

# Create the namespaces manually
kubectl create ns revcog-test
kubectl create ns revcog-prod
```

## Create Azure DevOps Project
Currently, creating new Azure DevOps Organizations through the CLI is not supported; this is done on the [Azure DevOps website](https://dev.azure.com/). Once an organization has been created, you can set it as the default organization for subsequent commands:
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

## Configure Azure DevOps
Add the [SonarCloud pipeline plugin](https://marketplace.visualstudio.com/items?itemName=SonarSource.sonarcloud) to your organization.

Although it is possible to create service connections with the CLI, it is a very involved process. It is far simpler to navigate to the Service Connections section of the Project Settings on the [Azure DevOps website](https://dev.azure.com/). For this pipeline, the following connections are required:
* a Docker Registry connection called `docker-cr-conn`
* a Kubernetes connection called `kube-conn`
* a SonarCloud connection called `sonar-conn`

Steps to configure Slack integration can be found [here](https://docs.microsoft.com/en-us/azure/devops/pipelines/integrations/slack?view=azure-devops).

## Create the Pipeline
If you do not specify the organization and project with `az devops configure`, you will need to specify those in the command to create the pipeline.

Use the following command to create the pipeline:
```sh
az pipelines create \
    --name 'MyPipeline' \
    --description 'Pipeline for MyRepo' \
    --repository https://github.com/MyOrg/MyRepo \
    --branch main \
    --yml-path azure-pipelines.yml
```