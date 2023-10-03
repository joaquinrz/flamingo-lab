# Harnessing Argo & Flux: The Quest to Scale Add-Ons Beyond 10k Clusters

![License](https://img.shields.io/badge/license-MIT-green.svg)

## Overview

Managing cluster add-ons at scale across private clouds, public clouds, and the edge presents a significant challenge. The intricate nature of large-scale operations can lead to inefficiencies, increased costs, and potential vulnerabilities if not managed effectively. This challenge can be addressed by harnessing the capabilities of Argo CD, Flux, and Flamingo to scale beyond 10,000 clusters while addressing the growing needs for scale, logging, and monitoring. We'll discuss how Flux and Flamingo contributes to the lifecycle of cluster add-ons at a massive scale and how the Argo CD API can be integrated into a cluster lifecycle solution.

This repository demonstrates a sample implementation on to integrate open-source tools such as ArgoCD, Flamingo and Flux.

## Prerequisites

For our sample will be using Azure Kubernetes Service (AKS). Before starting, you will need the following:

- An Azure account (already logged in with the Azure CLI)
- Azure CLI [download](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
- [Helm CLI](https://helm.sh)
- Optional but recommented (and the instructions below assume you have one) A working DNS zone in Azure, to use proper DNS names and automatic certifcates provisioning with Let's Encrypt

All of the applications will be installed via ArgoCD!

## Step 1: Create an ArgoCD/Flamingo management cluster with AKS

To create a new management cluster in AKS, run the following commands. Otherwise, if you already have an existing AKS cluster, you can skip this step and proceed to connecting to the existing AKS cluster. Change accoring to your liking, especially the `AZURE_DNS_ZONE`:

```bash
export AZURE_SUBSCRIPTION_ID=$(az account show --query id -o tsv)
az account set --subscription $AZURE_SUBSCRIPTION_ID

export CLUSTER_RG=management
export CLUSTER_NAME=management
export LOCATION=southcentralus
export IDENTITY_NAME=gitops$RANDOM
export NODE_COUNT=4
export AZ_AKS_VERSION=1.26.6
export AZURE_DNS_ZONE=kubelab.pro
export AZURE_DNS_ZONE_RESOURCE_GROUP=dns
```

Create a resource group for your AKS cluster with the following command, replacing <resource-group> with a name for your resource group and <location> with the Azure region where you want your resources to be located:

```bash
az group create --name $CLUSTER_RG --location $LOCATION
```

To use automatic DNS name updates via external-dns, we need to create a new managed identity and assign the role of DNS Contributor to the resource group containg the zone resource  

```bash
export IDENTITY=$(az identity create  -n $IDENTITY_NAME -g $CLUSTER_RG --query id -o tsv)
export IDENTITY_CLIENTID=$(az identity show -g $CLUSTER_RG -n $IDENTITY_NAME -o tsv --query clientId)

echo "Sleeping a bit (35 seconds) to let AAD catch up..."
sleep 35

export DNS_ID=$(az network dns zone show --name $AZURE_DNS_ZONE \
  --resource-group $AZURE_DNS_ZONE_RESOURCE_GROUP --query "id" --output tsv)

az role assignment create --role "DNS Zone Contributor" --assignee $IDENTITY_CLIENTID --scope $DNS_ID
```

Create an AKS cluster with the following command:

```bash
az aks create -k $AZ_AKS_VERSION -y -g $CLUSTER_RG \
    -s Standard_B4ms -c $NODE_COUNT \
    --assign-identity $IDENTITY --assign-kubelet-identity $IDENTITY \
    --network-plugin kubenet -n $CLUSTER_NAME
```

Connect to the AKS cluster:

```bash
az aks get-credentials --resource-group $CLUSTER_RG --name $CLUSTER_NAME
```

Verify that you can connect to the AKS cluster:

```bash
kubectl get nodes
```

## Step 2:  Install ArgoCD + Flamingo

ArgoCD is an open-source continuous delivery tool designed to simplify the deployment of applications to Kubernetes clusters. It allows developers to manage and deploy applications declaratively, reducing the amount of manual work involved in the process. With ArgoCD, developers can automate application deployment, configuration management, and application rollouts, making it easier to deliver applications reliably to production environments.

Flamingo - the Flux Subsystem for Argo is a solution that integrates Flux into Argo CD for a seamless GitOps experience with Kubernetes clusters. Flamingo is a drop-in and non-invasive component for any Argo CD users to explore today.

You can install ArgoCD + Flamingo on your Kubernetes cluster by running the following commands in your terminal or command prompt. These commands will download and install ArgoCD + Flamingo on your cluster, allowing you to use it for GitOps-based continuous delivery of your applications

> **_NOTE:_** Make sure to update the values for ingress hostname, and also your domain name in the various helm charts under `gitops/management` folder; Also make sure to update the external-dns values to include your tenantId and subscriptionId. we will update the README when we find a better way to dynamically inject these values into the helm charts deployed by ArgoCD

Add ArgoCD Helm Repo:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

Edit the `gitops/management/argocd/argocd-values.yaml` with your hostname and domain name for ArgoCD ingress, then install ArgoCD:

```bash
helm upgrade -i -n argocd \
  --version 5.46.7 \
  --create-namespace \
  --values gitops/management/argocd/argocd-values.yaml \
  argocd argo/argo-cd

helm upgrade -i -n argocd \
  --version 0.0.9\
  --create-namespace \
  --values argocd-initial-objects.yaml \
  argocd-apps argo/argocd-apps
```

Verify that ArgoCD is running:

```bash
kubectl get pods -n argocd
```

Access the ArgoCD web UI by running the following command, and then open the URL in a web browser (ingress, external-dns and cert-manager take care of certificates and DNS hostname resolution):

```bash
open https://argocd.$AZURE_DNS_ZONE
```

> **_NOTE:_** ArgoCD is in read-only mode for anonymous users, that should be enough to monitor the installaation progress, but if you want to change things, retrieve the secret with: 
> `kubectl get secret -n argocd argocd-initial-admin-secret  -o=jsonpath='{.data.password}'| base64 -D`

## Step 4:  Deploy clusters via Pull Requests

Open a PR against your main branch, modifying the appset-vcluster.yaml to deploy your ephemeral clusters!
