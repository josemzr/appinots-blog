---
title: "How to Deploy the Node Autoprovisioner in AKS"
date: 2025-01-15
draft: false
tags: ["AKS", "Kubernetes", "Autoscaling", "Node Autoprovisioner", "Azure"]
---

The **Node Autoprovisioner** is a powerful open-source component that enables Kubernetes to automatically provision (and deprovision) nodes based on pod scheduling needs. It's similar in intent to the [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md), but it allows for **greater flexibility** by dynamically creating **new node pools** with custom configurations — including taints, labels, and instance types.

This guide will walk you through deploying the Node Autoprovisioner on **Azure Kubernetes Service (AKS)**.


## Prerequisites

- An existing AKS cluster (version >= 1.26 recommended)
- Azure CLI installed and logged in
- Helm installed
- Cluster-admin access (`kubectl` configured)
- The AKS cluster must use **VMSS** (Virtual Machine Scale Sets)
- Enable **Managed Identity** on the cluster

## Enable AKS Preview Features (if needed)

Node Autoprovisioner requires enabling the AKS Preview CLI extension:

```bash
az extension add --name aks-preview
az extension update --name aks-preview
```

## Register the NodeAutoProvisioningFeature 


```bash
az feature register --namespace "Microsoft.ContainerService" --name "NodeAutoProvisioningPreview"

#Verify status
az feature show --namespace "Microsoft.ContainerService" --name "NodeAutoProvisioningPreview"
```

> ⚠️ Note: This may take a few minutes to propagate.

## Create a new AKS cluster with NAP enabled

```bash
az aks create \
    --name $CLUSTER_NAME \
    --resource-group $RESOURCE_GROUP_NAME \
    --node-provisioning-mode Auto \
    --network-plugin azure \
    --network-plugin-mode overlay \
    --network-dataplane cilium \
    --generate-ssh-keys
```

If you want to use an existing cluster, you can update it:

```bash
az aks update --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP_NAME --node-provisioning-mode Auto --network-plugin azure --network-plugin-mode overlay --network-dataplane cilium
```


## Create a new NodePool

You can configure a new NodePool:

```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: Never
  template:
    spec:
      nodeClassRef:
        name: default

      # Requirements that constrain the parameters of provisioned nodes.
      # These requirements are combined with pod.spec.affinity.nodeAffinity rules.
      # Operators { In, NotIn, Exists, DoesNotExist, Gt, and Lt } are supported.
      # https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#operators
      requirements:
      - key: kubernetes.io/arch
        operator: In
        values:
        - amd64
      - key: kubernetes.io/os
        operator: In
        values:
        - linux
      - key: karpenter.sh/capacity-type
        operator: In
        values:
        - on-demand
      - key: karpenter.azure.com/sku-family
        operator: In
        values:
        - D
```

Apply with:

```bash
kubectl apply -f nodepool.yaml
```

## ✅ Step 4: Test the Autoprovisioning

Deploy a workload that exceeds your current node pool capacity:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-workload
spec:
  replicas: 10
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: "2"
```

Watch how new node pools are dynamically created:

```bash
kubectl get nodepools
kubectl get nodes -w
```

## Notes and Tips

- Make sure your AKS service principal or managed identity has `Contributor` access to the **node resource group**.
- You can combine this with **taints and tolerations** for workload isolation.

## References

- [Node Autoprovisioner GitHub](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler/provisioner)
- [Azure AKS Documentation](https://learn.microsoft.com/en-us/azure/aks/)
- [Kubernetes Autoscaling Concepts](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
