# TOS CICD TERRAFORM

![image](https://github.com/AlixBnd/TOS-CICD-Terraform/assets/137909386/a58318c9-8bb6-47aa-b476-6f00afa55f0e)


Qu'est-ce que Terraform ? Terraform est un logiciel d'infrastructure as code multicloud. Si vous utilisez ce dernier, il est possible de déployer votre infrastructure dans Azure grâce au CICD d'Azure DevOps.

# Pré-requis

 - Compréhension basique de Terraform
 - Compréhension basique d'Azure et Azure DevOps
 - Extensions Azure DevOps
	 -  Azures Pipelines Terraform Task by Jason Johnson
	 - Terraform by Microsoft Labs
 
 
Pour acquérir une compréhension basique de Terraform, je vous invite à suivre le tutoriel proposé sur le site officiel :

https://developer.hashicorp.com/terraform/tutorials/azure-get-started

## Fichiers nécessaires au déploiement

Dans votre repo Azure DevOps, vous aurez besoin d'une architecture similaire :

Terraform
|
|---> main.tf
|---> variables.tf

 1. main.tf
 

  ```hcl
terraform {
    required_version = ">= 1.1.0"
    
    backend "azurerm" {
    resource_group_name = "backend"
    storage_account_name = "backendtestttt1123"
    container_name = "backendcontainer"
    key = "tf/terraform.tfstate"
    }

    required_providers {
    azurerm = {
    source = "hashicorp/azurerm"
    version = "~> 3.0.2"
    }
    }
}

provider "azurerm" {
    features {}
    
    client_id = var.ARM_CLIENT_ID
    client_secret = var.ARM_CLIENT_SECRET
    tenant_id = var.ARM_TENANT_ID
    subscription_id = var.ARM_SUBSCRIPTION_ID
}
```
    
 Le bloc backend "azurerm" correspond à l'endroit où sera stocké le tfstate, ici dans un blob storage sur Azure.
 Quant au bloc provider "azurerm", ses arguments attribuent aux variables les valeurs générées précédemment dans le [tutoriel](https://developer.hashicorp.com/terraform/tutorials/azure-get-started/azure-build) et qui seront stockées dans l'Azure DevOps.

Plus bas dans le fichier, créer les ressources dont vous avez besoin comme vu dans le [tutoriel](https://developer.hashicorp.com/terraform/tutorials/azure-get-started/azure-build)

*Note: Il est recommandé de créer un nouveau groupe de ressource et de ne pas utiliser celui du backend.*

 2. variables.tf

Ces variables sont obligatoirement définies dans le fichier.

 

       variable  "ARM_CLIENT_SECRET" {
        description = "The Client Secret for the Service Principal."
        type = string
        }
        
        variable  "ARM_TENANT_ID" {
        description = "The Tenant ID for the Service Principal."
        type = string
        }
        
        variable  "ARM_SUBSCRIPTION_ID" {
        description = "The Subscription ID where resources are deployed."
        type = string
        }
        
        variable  "ARM_CLIENT_ID" {
        description = "The Subscription ID where resources are deployed."
        type = string
        }

Si la définition n'est pas faite, les variables seront inaccessibles à Terraform.   

## Azure DevOps

 1. Ajout des variables

Pour ajouter les variables nécessaires au déploiement de l'infrastructure, nous allons créer un groupe de variables.

![image](https://github.com/AlixBnd/TOS-CICD-Terraform/assets/137909386/13869bd1-cf5f-4808-a654-a4413384316b)


Le résultat doit être similaire à ci-dessous, sans oublier de **fermer** les petits cadenas pour ne pas laisser la valeur en clair.

![image](https://github.com/AlixBnd/TOS-CICD-Terraform/assets/137909386/184bfb04-cdbf-4ddc-8ff8-a24a6ccfe494)


Ajouter ensuite la pipeline dans les permissions de cette manière

![image](https://github.com/AlixBnd/TOS-CICD-Terraform/assets/137909386/bce3d4ed-7ef8-4529-a245-572af478d617)


Il est aussi possible d'autoriser l'accès aux variables à tout le projet de la manière suivante : 

![image](https://github.com/AlixBnd/TOS-CICD-Terraform/assets/137909386/1c787ebf-4ac1-45a7-83ec-21c7c9701d76)


 2. Configuration de la pipeline

Dans la section "variables" de la pipeline, lier le groupe de variables créé précédemment, elles seront donc accessibles à la pipeline.

![image](https://github.com/AlixBnd/TOS-CICD-Terraform/assets/137909386/06a98ad8-7e11-461d-9f0a-acd1517c809c)


Les tâches nécessaires sont les suivantes :

![image](https://github.com/AlixBnd/TOS-CICD-Terraform/assets/137909386/7dd5a2f0-3586-4ad0-a99f-72e5eb15443b)


Dans la première tache Command Line Script, cette commande sera exécutée afin de se connecter à son compte Azure :

    az login --service-principal -u "$(ARM_CLIENT_ID)" -p "$(ARM_CLIENT_SECRET)" --tenant "$(ARM_TENANT_ID)"

Dans la seconde tâche, cette commande sera exécutée et sera primordiale pour que Terraform puisse utiliser les variables de la pipeline dans les différentes étapes :

    echo "##vso[task.setvariable variable=TF_VAR_ARM_CLIENT_ID]$(ARM_CLIENT_ID)"
    echo "##vso[task.setvariable variable=TF_VAR_ARM_CLIENT_SECRET]$(ARM_CLIENT_SECRET)"
    echo "##vso[task.setvariable variable=TF_VAR_ARM_TENANT_ID]$(ARM_TENANT_ID)"    
    echo "##vso[task.setvariable variable=TF_VAR_ARM_SUBSCRIPTION_ID]$(ARM_SUBSCRIPTION_ID)"

L'étape *Use Terraform Latest* aura pour but d'utiliser la dernière version de Terraform dans le CLI.

Le *Terraform Init* est l'une des étapes la plus importante, il ne faut donc pas louper sa configuration.

**Premièrement**, ne pas oublier de spécifier le bon configuration directory "terraform" si l'arborescence présentée à la première étape est respectée. Cela rendra accessibles les fichiers de Terraform.

Ensuite, ne pas oublier de configurer le **AzureRM Backend Configuration**.
Ce dernier permettra de créer le backend qui stockera le fichier tfstate dans un blob.

**Attention** chacune des valeurs doivent correspondre à celles indiquée dans la section backend de votre main.

*Note: Le chemin indiqué dans Key sera celui où le fichier tfstate sera situé dans le blob*

![image](https://github.com/AlixBnd/TOS-CICD-Terraform/assets/137909386/b8a94573-4beb-4e00-89e8-3d8ff877a8d2)

![image](https://github.com/AlixBnd/TOS-CICD-Terraform/assets/137909386/5dd2835d-c814-4133-b426-3d2e72c6588e)


Ensuite, pour *fmt* et *validate*, il suffit de spécifier le configuration directory.
Cependant, pour l'étape *apply*, spécifier le directory ainsi que le provider comme ci-dessous :

![image](https://github.com/AlixBnd/TOS-CICD-Terraform/assets/137909386/c6de8831-520a-4410-86a2-72db49f39d70)


Vous pouvez désormais lancer votre pipeline ✅

![image](https://github.com/AlixBnd/TOS-CICD-Terraform/assets/137909386/4993dd08-33d4-4699-b363-06d509fc907f)




**Félicitations, vous avez déployé votre infrastructure avec succès**  🥳



