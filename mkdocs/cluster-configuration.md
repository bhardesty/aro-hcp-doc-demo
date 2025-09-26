
# Overview of cluster configuration choices

This article explains the main cluster configuration choices you can make when creating an Azure Red Hat OpenShift with hosted control planes cluster. By setting these options, you can customize the cluster to meet your needs.

Many of these options can't be changed after the cluster is created. If you need to change any of these options, you must create a new cluster.

You can configure the following cluster options:

* Cluster name and location
* OpenShift version
* Domain prefix
* Network configuration - Classless Inter-Domain Routing (CIDR) ranges
* Cluster autoscaling
* etcd encryption
* Node drain grace period
* Image registry

In addition, you can configure the following node pool properties:

* Node pool name and location
* OpenShift version
* Platform
* Number of nodes to provision (note: autoscaling and replicas)
* Labels
* Taints
* Node drain grace period

## Understand managed identities in Azure Red Hat OpenShift with hosted control planes

Managed identities are service principals of a special type, which can be used to control or enable access to Azure resources. In Azure Red Hat OpenShift with hosted control planes, the cluster Operators use managed identities to obtain the temporary permissions required to carry out cluster operations, such as managing back-end storage, cloud provider credentials, and external access to a cluster.

For more information about OpenShift cluster Operators, see [Cluster Operators](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html-single/architecture/index#cluster-operators_control-plane) in the OpenShift Container Platform documentation.

For more information about managed identities, see [Understand managed identities in Azure Red Hat OpenShift (preview)](https://learn.microsoft.com/en-us/azure/openshift/howto-understand-managed-identities).

### Required managed identities

The following OpenShift cluster Operators require managed identities:

* OpenShift Cluster API Provider  
* OpenShift Control Plane Operator  
* OpenShift Cloud Controller Manager  
* OpenShift Cluster Ingress Operator  
* OpenShift Disk Storage Operator (requires two managed identities: one for the control plane, and one for the data plane)  
* OpenShift File Storage Operator (requires two managed identities: one for the control plane, and one for the data plane)  
* OpenShift Image Registry Operator (requires two managed identities: one for the control plane, and one for the data plane)  
* OpenShift Network Operator

In addition, the Azure Red Hat OpenShift Hosted Control Planes Service also requires a user-assigned managed identity to enable the managed identities for the cluster Operators, and to create federated credentials for them.

### Managed identity role assignments

Each of the required managed identities must be associated with a built-in Azure role. Roles grant the required permissions to the supporting managed identities to enable cluster operations.

The following table lists the required role assignment for each managed identity, and the scope at which the role grants permission to the managed identity.

| Managed identity | Role | Scope |
| :---- | :---- | :---- |
| OpenShift Cluster API Provider | Azure Red Hat OpenShift Hosted Control Planes Cluster API Provider. This role enables permissions to allow the cluster API to manage nodes, networks, and disks for the Azure Red Hat OpenShift with hosted control planes cluster. | Subnet |
| OpenShift Hosted Control Planes Control Plane Operator | Azure Red Hat OpenShift Hosted Control Planes Control Plane Operator. This role enables the Control Plane Operator to read resources necessary for the OpenShift cluster. | Subnet and Network Security Group |
| OpenShift Cloud Controller Manager | [Azure Red Hat OpenShift Cloud Controller Manager](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles/containers#azure-red-hat-openshift-cloud-controller-manager) | Subnet and Network Security Group |
| OpenShift Cluster Ingress Operator | [Azure Red Hat OpenShift Cluster Ingress Operator](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles/containers#azure-red-hat-openshift-cluster-ingress-operator) | Subnet |
| OpenShift Disk Storage Operator (control plane) | none | n/a |
| OpenShift Disk Storage Operator (data plane) | none | n/a |
| OpenShift File Storage Operator (control plane) | [Azure Red Hat OpenShift File Storage Operator](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles/containers#azure-red-hat-openshift-file-storage-operator) | Subnet and Network Security Group |
| OpenShift File Storage Operator (data plane) | [Azure Red Hat OpenShift File Storage Operator](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles/containers#azure-red-hat-openshift-file-storage-operator) | Subnet and Network Security Group |
| OpenShift Image Registry Operator (control plane) | none | n/a |
| OpenShift Image Registry Operator (data plane) | none | n/a |
| OpenShift Network Operator | [Azure Red Hat OpenShift Network Operator](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles/containers#azure-red-hat-openshift-network-operator) | Vnet and Subnet |
| OpenShift Hosted Control Planes Service | Azure Red Hat OpenShift Hosted Control Planes Service. This role can perform the following actions: <ul><li>`Microsoft.Network/virtualNetworks/read`</li><li>`Microsoft.Network/virtualNetworks/subnets/read`</li><li>`Microsoft.Network/virtualNetworks/subnets/write`</li><li>`Microsoft.Network/routeTables/read`</li><li>`Microsoft.Network/routeTables/join/action`</li><li>`Microsoft.Network/natGateways/read`</li><li>`Microsoft.Network/natGateways/join/action`</li><li>`Microsoft.Network/networkSecurityGroups/read`</li><li>`Microsoft.Network/networkSecurityGroups/join/action`</li></ul> | Vnet, Subnet, and Network Security Group |
