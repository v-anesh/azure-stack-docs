---
title: Deploy Kubernetes cluster to custom virtual network on Azure Stack Hub  
description: Learn how to deploy a Kubernetes cluster to a custom virtual network on Azure Stack Hub.
author: mattbriggs

ms.topic: article
ms.date: 04/23/2021
ms.author: mabrigg
ms.reviewer: waltero
ms.lastreviewed: 04/23/2021

# Intent: As an Azure Stack Hub user, I would like to deploy a Kubernetes cluster using the AKS engine on a custom virtual network so that I can deliver my service in an environment that extends my data center or in a hybrid cloud solution with my cluster in Azure Stack Hub and Azure.
# Keywords: virtual network ASK engine Azure Stack Hub

---

# Deploy a Kubernetes cluster to a custom virtual network on Azure Stack Hub 

You can deploy a Kubernetes cluster using the Azure Kubernetes Service (AKS) engine on a custom virtual network. This article looks at finding the information you need in your virtual network. You can find steps for calculating the IP addresses used by your cluster, setting the vales in the API Model, and setting the route table and network security group.

The Kubernetes cluster in Azure Stack Hub using the AKS engine uses the kubenet network plugin. The AKS engine on Azure Stack Hub also supports the Azure CNI network plugin.  
  - For a discussion of the kubenet networking plugin in Azure, see [Use kubenet networking with your own IP address ranges in Azure Kubernetes Service (AKS)](/azure/aks/configure-kubenet).
  - For a discussion of the Azure CNI networking plugin in Azure, see [Configure Azure CNI networking in Azure Kubernetes Service (AKS)](/azure/aks/configure-azure-cni).

## Constraints when creating a custom virtual network

-  The custom VNET must be in the same subscription as all of the other components of the Kubernetes cluster.
-  The pool of master nodes and the pool of agent nodes must be in the same virtual network. You can deploy your nodes into different subnets within the same virtual network.
-  The Kubernetes cluster subnet must use an IP range within the space of the custom virtual network IP range, see [Get the IP address block](#get-the-ip-address-block).

## Create custom virtual network

You must have a custom virtual network in your Azure Stack Hub instance. For more information, see [Quickstart: Create a virtual network using the Azure portal](/azure/virtual-network/quick-create-portal).

Create a new subnet in your virtual network. You will need to the get the subnet Resource ID and IP address range. You will use the Resource ID and range in your API model when you deploy your cluster.

1. Open the Azure Stack Hub user portal in your Azure Stack Hub instance.
2. Select **All resources**.
3. Enter the name of your virtual network in the search box.
4. Select **Subnets** > **+ Subnets** to add a subnet.
5. Add a **Name** and an **Address range** using CIDR notation. Select **OK**.
4. Select **Properties** in the **Virtual networks** blade. Copy the **Resource ID**, and then add `/subnets/<nameofyoursubnect>`. You will use this value as your value for the `vnetSubnetId` key in the API model for your cluster. The Resource ID for the subnet uses the following format:<br>`/subscriptions/SUB_ID/resourceGroups/RG_NAME/providers/Microsoft.Network/virtualNetworks/VNET_NAME/subnets/SUBNET_NAME`

    ![virtual network Resource ID](media/kubernetes-aks-engine-custom-vnet/virtual-network-id.png)

5. Select **Subnets** in the **Virtual networks** blade. Select the subnet name, for example default.
    
    ![virtual network CIDR block](media/kubernetes-aks-engine-custom-vnet/virtual-network-cidr-block.png)
    
6. In the subnet blade, make a note of the address range and the virtual network CIDR Block, for example: `10.1.0.0 - 10.1.0.255 (256 addresses)` and `10.1.0.0/24`.

## Considerations for selecting an address space

When you create a custom virtual network, you specify the IP address space of your network and an IP address range for every subnet. Consider the following factors when you choose the address spaces and ranges to use in your Kubernetes cluster:
-  Overlapping address spaces might result in IP address clashes or communication errors. To reduce the risk of overlapping IP addresses, choose a unique address space for your new virtual network.
-  Address spaces in the `10.` and `172.` ranges often are used for private networks, and they might be used by your existing datacenter infrastructure. If your Kubernetes applications use resources in your datacenter, reduce the risk of clashes by choosing an address space for your custom virtual network that's different from your datacenter's address space.
-  We recommend that you use a dedicated subnet for your Kubernetes cluster.

## Get the IP address block

The AKS engine supports deploying into an existing virtual network. When deploying into an existing subnet, your cluster will use a block of consecutive IPs for agents and another for masters.

You will need to set two values. You will need to know the number of IP addresses you will need to reserve for your cluster, and the first consecutive static IP within the subnet IP space.

The AKS engine requires a range of up to 16 unused IP addresses when you use multiple master nodes. The cluster will use one IP address for each master up to five masters. The AKS engine will also require the next 10 IP address after the last master for headroom IP address reservation. Finally, another IP address will be used by the load balancer after the masters and headroom reservation for a total of 16.

When placing your block of IP addresses, the subnet requires the following allocations of the existing IP addresses:
 - The first four IP addresses and the last IP address are reserved and can't be used in any Azure subnet
 - A buffer of 16 IP addresses should be left open.
 - The value of your cluster's first IP should be toward the end of the address space to avoid IP conflicts. If possible, assign to the `firstConsecutiveStaticIP` property to an IP address near the *end* of the available IP address space in the subnet.

In the following example, you can see how these various considerations fill out the IP range in a subnet. This is for three masters. If you are using a subnet with 256 addresses, for example 10.1.0.0/24, you will need to set your first consecutive static IP address at 207. The following table shows the addresses and considerations:

| Range for /24 subnet | Number | Note |
| --- | --- | --- |
| 10.1.0.0  - 10.1.03 | 4 | Reserved in Azure subnet. |
| **10.1.0.224**-10.1.0.238 | 14 | IP address count for an AKS engine defined cluster.<br><br> 3 IP addresses for 3 masters<br>10 IP addresses for headroom<br>1 IP address for the load balancer |
| 10.1.0.239 - 10.1.0.255 | 16 | 16 IP address buffer. |
| 10.1.0.256 | 1 | Reserved in Azure subnet. |

In this example, then `firstConsecutiveStaticIP` property would be `10.1.0.224`.

For larger subnets, for example /16 with more than 60 thousand addresses, you may not find it to be practical to set your static IP assignments to the end of the network space. Set your cluster static IP address range away from the first 24 addresses in your IP space so that the cluster can be resilient when claiming addresses.

## Update the API model

Update the API model used to deploy the cluster from the AKS engine to your custom virtual network.

In **masterProfile** set the following values:

| Field | Example | Description |
| --- | --- | --- |
| vnetSubnetId | `/subscriptions/77e28b6a-582f-42b0-94d2-93b9eca60845/resourceGroups/MDBN-K8S/providers/Microsoft.Network/virtualNetworks/MDBN-K8S/subnets/default` | Specify the Resource ID the subnet.  |
| firstConsecutiveStaticIP | 10.1.0.224 | Assign to the `firstConsecutiveStaticIP` configuration property an IP address that is near the *end* of the available IP address space in the desired subnet. `firstConsecutiveStaticIP` only applies to the master pool. |

In **agentPoolProfiles** set the following values:

| Field | Example | Description |
| --- | --- | --- |
| vnetSubnetId | `/subscriptions/77e28b6a-582f-42b0-94d2-93b9eca60845/resourceGroups/MDBN-K8S/providers/Microsoft.Network/virtualNetworks/MDBN-K8S/subnets/default` | Specify the Azure Resource Manager path ID the subnet.  |

In **orchestratorProfile**, find **kubernetesConfig** and set the following value:

| Field | Example | Description |
| --- | --- | --- |
| clusterSubnet | `10.244.0.0/16` | The IP subnet used for allocating IP addresses for pod network interfaces. The subnet must be in the VNET address space. With Azure CNI enabled, the default value is 10.240.0.0/12. Without Azure CNI, the default value is 10.244.0.0/16.  the cluster subnet (POD network) network should not use the same IP space as that of the VNET. Use /16 instead /24 subnet. If you use /24, this subnet will be assigned to one node only. Other node will not get POD network assigned, as you will have run out of the IP space, so they will be not ready in the cluster. |

For example:

```json
"masterProfile": {
  ...
  "vnetSubnetId": "/subscriptions/77e28b6a-582f-42b0-94d2-93b9eca60845/resourceGroups/MDBN-K8S/providers/Microsoft.Network/virtualNetworks/MDBN-K8S/subnets/default",
  "firstConsecutiveStaticIP": "10.1.0.224",
  ...
},
...
"agentPoolProfiles": [
  {
    ...
    "vnetSubnetId": "/subscriptions/77e28b6a-582f-42b0-94d2-93b9eca60845/resourceGroups/MDBN-K8S/providers/Microsoft.Network/virtualNetworks/MDBN-K8S/subnets/default",
    ...
  },
    ...
"kubernetesConfig": [
  {
    ...
    "clusterSubnet": "172.16.244.0/16",
    ...
  },

```

## Deploy your cluster

After adding the values to your API model, you can deploy your cluster from your client machine using the `deploy` command using the AKS engine. For instructions, see [Deploy a Kubernetes cluster](azure-stack-kubernetes-aks-engine-deploy-cluster.md#deploy-a-kubernetes-cluster).

## Set the route table

After you deploy your cluster, return to your virtual network in the Azure Stack user portal. Set both the route table and the network security group (NSG) in the subnet blade. If you're not using Azure CNI, for example, `networkPlugin`: `kubenet` in the `kubernetesConfig` API model configuration object. After you have successfully deployed a cluster to your custom virtual network, get the ID of the Route Table resource from **Network** blade in your cluster's resource group.

1. Open the Azure Stack Hub user portal in your Azure Stack Hub instance.
2. Select **All resources**.
3. Enter the name of your virtual network in the search box.
4. Select **Subnets** and then select the name of the subnet that contains your cluster.
    
    ![route table and network security group](media/kubernetes-aks-engine-custom-vnet/virtual-network-route-table.png)
    
5. Select **Route table** and then select the route table for your cluster.

> [!NOTE]  
> Custom virtual network for Kubernetes Windows cluster has a [known issue](https://github.com/Azure/aks-engine/issues/371).

## Next steps

- Read about the [The AKS engine on Azure Stack Hub](azure-stack-kubernetes-aks-engine-overview.md)  
- Read about [Azure Monitor for containers overview](/azure/azure-monitor/insights/container-insights-overview)
