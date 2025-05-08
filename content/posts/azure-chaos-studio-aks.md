---
title: "Using Chaos Studio with AKS"
date: 2025-02-04
draft: false
tags: ["AKS", "Azure", "Chaos Engineering", "Chaos Studio", "Resilience"]
---

**Azure Chaos Studio** is a managed chaos engineering service that allows you to introduce controlled faults into your Azure environment to validate resilience and robustness. In this post, you'll learn how to use Chaos Studio to run experiments on **Azure Kubernetes Service (AKS)** clusters.

---


## Prerequisites

- An existing AKS cluster
- Azure CLI installed and logged in
- `kubectl` configured
- AKS cluster must be running in a supported region
- Chaos Studio enabled in your subscription

---

## Register Required Resource Providers

```bash
az provider register --namespace Microsoft.Chaos
az provider register --namespace Microsoft.ContainerService
```

---

## Enable Chaos Studio on Your AKS Cluster

```bash
az chaos target create --resource-id $(az aks show -g <resource-group> -n <aks-name> --query id -o tsv) \
  --target-type Microsoft-AzureKubernetesServiceCluster \
  --location <region>
```

You can also enable Chaos Studio from the **Azure Portal** under your AKS resource > **Chaos Studio** > **Enable**.

---

## Create a Chaos Experiment

You can define a chaos experiment using an ARM template or with the Azure CLI.

Example experiment (restart AKS pods):

```bash
az chaos experiment create --name aks-restart-test \
  --resource-group <resource-group> \
  --location <region> \
  --identity-type SystemAssigned \
  --selectors selector-id="mySelector",type="List",targets="aks-resource-id" \
  --steps "name=restart-pods,actions='type=continuous-reboot'" \
  --duration PT5M
```

Or define it as a JSON file and apply it using:

```bash
az chaos experiment create --from-file aks-chaos-experiment.json
```

---

## Run the Experiment

```bash
az chaos experiment start --name aks-restart-test --resource-group <resource-group>
``` 

You can monitor the experiment progress in the Azure Portal or via:

```bash
az chaos experiment show --name aks-restart-test --resource-group <resource-group>
```

---

Chaos engineering is not about breaking things — it's about learning how your system breaks, and building confidence in its ability to recover. Start small, iterate, and build trust in your architecture.
