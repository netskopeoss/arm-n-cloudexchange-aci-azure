# Introduction

This Infrastructure as Code (IaC) will help you deploy Netskope Cloud Exchange in your Azure environment and configure the following;

- Create a virtual network and a subnet to deploy container instances for cloud exchange. By deploying container groups into an Azure virtual network, your containers will communicate securely with other resources in the virtual network.
- Create a Container group to run cloud exchange containers, this container group is a collection of four containers and will share a lifecycle, resources, local network, and storage volumes.
- Create four Azure file shares as volume mounts and each container mounts one of the shares locally.
- Use your own Azure container registry to pull down dokcer images.
- Create a network security group with required rules and associate with the subnet. The network security rule is configure to allow communication to cloud exchange admin UI to the supplied trusted IP. i.e. parameter `var.trustedIP `.
- Provide Netskope Cloud Exchange IP address as template output value.

## Architecture Diagram

This IaC will create the Azure resources as shown below.

![](.//images/ce-acr-aci-azure-n.png)

*Fig 1. Netskope Cloud Exchange deployment in Azure using Azure Container Instances*

## Deployment

To deploy this template in Azure:

- You need either Azure PowerShell or Azure Command-Line Interface (CLI) to deploy the template. If you use Azure CLI, you need to have version 2.37.0 or later. If you're using Azure PowerShell, make sure you have version 7.2.4 or later. The instructions provided below are using Azure CLI.

- Clone the GitHub repository for this deployment.

- Login with `az login` command. 

- If you have multiple Azure subscriptions, choose the subscription you want to use. Replace SubscriptionName with your subscription name. You can also use your subscription ID instead of your subscription name.

```sh 
az account set --subscription SubscriptionName

```

- When you deploy a template, you can specify a resource group to contain the resources. Before running the deployment command, create the resource group. For example;

```sh
az group create --name netskope-rg --location "canadacentral"

```

- Download netskope cloud exchange containers from Netskope's docker hub registery using the following commands.

``` sh
docker pull netskopetechnicalalliances/cloudexchange:ui3-latest
docker pull netskopetechnicalalliances/cloudexchange:core3-latest
docker pull netskopetechnicalalliances/azure:rabbitmq
docker pull netskopetechnicalalliances/azure:mongodb-5.0

```

- Login to Azure container registry, tag images and then upload netskope cloud exchange containers to your Azure container registery (ACR). Replace `<name-of-your-acr>` with your Azure Container registry name for example; `nsacr.azurecr.io`

``` sh
az acr login --name <name-of-your-acr>
docker tag netskopetechnicalalliances/cloudexchange:ui3-latest <name-of-your-acr>/cloudexchange:ui3-latest
docker tag netskopetechnicalalliances/cloudexchange:core3-latest <name-of-your-acr>/cloudexchange:core3-latest
docker tag netskopetechnicalalliances/azure:rabbitmq <name-of-your-acr>/cloudexchange:rabbitmq
docker tag netskopetechnicalalliances/azure:mongodb-5.0 <name-of-your-acr>/cloudexchange:mongodb-5.0
docker push <name-of-your-acr>/cloudexchange:ui3-latest
docker push <name-of-your-acr>/cloudexchange:core3-latest
docker push <name-of-your-acr>/cloudexchange:rabbitmq
docker push <name-of-your-acr>/cloudexchange:mongodb-5.0

```

- Rename the file `ce-parameters.json.example` to `ce-parameters.json` and specify values for the parameters as needed. This parameter file have few parameters already added but more parameters can be added to this file, if needed. Rather than passing parameters as inline values in the template file itself, consider using this parameters file to set parameter values. In this parameter file, the first detail to notice is the name of each parameter. The parameter names in the parameter file must match the parameter names in the template.

- To deploy the template, use the resource group you created earlier. Give a name to the deployment so you can easily identify it in the deployment history. For example;

``` sh
az deployment group create --name aci-ce --resource-group netskope-rg --template-file  ce-template.json --parameters '@ce-parameters.json'

```

- The deployment command returns results. Look for ProvisioningState to see whether the deployment succeeded. You can also verify the deployment by exploring the resource group from the Azure portal.


- The Cloud Exchange can now be accessed using it's Private IP address, provided as an output value of the template, either through a Jump/Bastion Host, through the private connectivity (e.g. ExpressRoute or Site to Site Tunnel) or if Netskope Private Access (NPA) is available then through the NPA. When first installed, Cloud Exchange does not require an SSL certificate and the Admin UI can be reached over an unencrypted connection. We recommend to install a private SSL certificate on the netskope cloud exchange. You can use let's encrypt a nonprofit Certificate Authority providing TLS certificates in the absence of any inhouse enterprise CA or commercial public CA.

- To install your own SSL certificate; 
    - Whitelist the IP address from where you will be uploading the SSL certificates to the data volume. 
    - Upload your own SSL Certificate and associated SSL private key to azure file share named `ssl-certs` mounted as a data volume for the ui container. The name of the SSL certificate and SSL Private key must be `cte_cert.cer` and `cte_cert_key.key`
    - Restart Container instances.

## Destruction

- To destroy this deployment, find the resource group in the Azure portal and select delete resource group from the top menu.

## Support

Netskope-provided scripts in this and other GitHub projects do not fall under the regular Netskope technical support scope and are not supported by Netskope support services.