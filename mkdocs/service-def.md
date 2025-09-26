
# Microsoft Azure Red Hat OpenShift with hosted control planes service definition

This documentation outlines the service definition for the Microsoft Azure Red Hat OpenShift with hosted control planes managed service.

## Account management

This section provides information about the service definition for Azure Red Hat OpenShift with hosted control planes account management.

### Billing and pricing

Azure Red Hat OpenShift with hosted control planes is billed directly to your Azure account. Azure Red Hat OpenShift with hosted control planes pricing is consumption based, with annual commitments or three-year commitments for greater discounting. The total cost of Azure Red Hat OpenShift with hosted control planes consists of two components:

* Azure Red Hat OpenShift with hosted control planes service fees  
* Azure infrastructure fees

Visit the [Microsoft Azure Red Hat Openshift pricing page](https://azure.microsoft.com/en-us/pricing/details/openshift/) for more details. 

### Cluster self-service

Customers can self-service their clusters, including, but not limited to:

* Create a cluster  
* Delete a cluster  
* Add or remove an identity provider  
* Add or remove a user from an elevated group  
* Configure cluster privacy  
* Add or remove node pools and configure autoscaling

You can perform these self-service tasks using the Microsoft Azure Red Hat Openshift APIs, Azure CLI, or Azure Portal. 

**Additional resources**

* [Red Hat Operator support](#red-hat-operator-support)

### Azure compute types

All Azure Red Hat OpenShift with hosted control planes clusters require a minimum of 2 worker nodes. Shutting down the underlying infrastructure through the cloud provider console is unsupported and can lead to data loss.

!!! note
    Approximately one vCPU core and 1 GiB of memory are reserved on each worker node and removed from allocatable resources. This reservation of resources is necessary to run processes required by the underlying platform. These processes include system daemons such as udev, kubelet, and container runtime among others. The reserved resources also account for kernel reservations. OpenShift Container Platform core systems such as audit log aggregation, metrics collection, DNS, image registry, SDN, and others might consume additional allocatable resources to maintain the stability and maintainability of the cluster. The additional resources consumed might vary based on usage. For additional information, see the [Kubernetes documentation](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#system-reserved).

**Additional resources**

* For a detailed listing of supported compute types, see [Sizes for virtual machines in Azure](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/overview?tabs=breakdownseries%2Cgeneralsizelist%2Ccomputesizelist%2Cmemorysizelist%2Cstoragesizelist%2Cgpusizelist%2Cfpgasizelist%2Chpcsizelist).

### Regions and availability zones

The following Azure regions are currently available for Azure Red Hat OpenShift with hosted control planes:

* UK South  
* Canada Central  
* Australia East  
* Switzerland North  
* Brazil South  
* Central India

!!! warning
    The region cannot be changed after a cluster has been deployed.

### Service Level Agreement (SLA)

Any SLAs for the service itself are defined in Appendix 4 of the [Red Hat Enterprise Agreement Appendix 4 (Online Subscription Services)](https://www.redhat.com/en/about/appendices).

### Limited support status

When a cluster transitions to a *Limited Support* status, Red Hat no longer proactively monitors the cluster, the SLA is no longer applicable, and credits requested against the SLA are denied. It does not mean that you no longer have product support. In some cases, the cluster can return to a fully-supported status if you remediate the violating factors. However, in other cases, you might have to delete and recreate the cluster.

A cluster might move to a Limited Support status for many reasons, including the following scenarios:

**If you remove or replace any native Azure Red Hat OpenShift with hosted control planes components or any other component that is installed and managed by Red Hat**  
If cluster administrator permissions were used, Red Hat is not responsible for any of your or your authorized users' actions, including those that affect infrastructure services, service availability, or data loss. If Red Hat detects any such actions, the cluster might transition to a Limited Support status. Red Hat notifies you of the status change and you should either revert the action or create a support case to explore remediation steps that might require you to delete and recreate the cluster.

If you have questions about a specific action that might cause a cluster to move to a Limited Support status or need further assistance, open a support ticket.

### Support

Azure Red Hat OpenShift with hosted control planes includes Red Hat Premium Support, which can be accessed by using the [Red Hat Customer Portal](https://access.redhat.com/support?extIdCarryOver=true&sc_cid=701f2000001Css5AAC).

See the Red Hat [Production Support Terms of Service](https://access.redhat.com/support/offerings/production/sla) for support response times.

Azure support is subject to a customer's existing support contract with Azure.

## Monitoring

This section provides information about the service definition for Azure Red Hat OpenShift with hosted control planes monitoring.

### Cluster metrics

Azure Red Hat OpenShift with hosted control planes clusters come with an integrated Prometheus stack for cluster monitoring including CPU, memory, and network-based metrics. This is accessible through the web console. These metrics also allow for horizontal pod autoscaling based on CPU or memory metrics provided by an Azure Red Hat OpenShift with hosted control planes user.

### Cluster notifications

Cluster notifications are messages about the status, health, or performance of your cluster.

Cluster notifications are the primary way that Red Hat Site Reliability Engineering (SRE) communicates with you about the health of your managed cluster. SRE may also use cluster notifications to prompt you to perform an action in order to resolve or prevent an issue with your cluster.

Cluster owners and administrators must regularly review and action cluster notifications to ensure clusters remain healthy and supported.

You can view cluster notifications in the Red Hat Hybrid Cloud Console, in the **Cluster history** tab for your cluster. By default, only the cluster owner receives cluster notifications as emails. If other users need to receive cluster notification emails, add each user as a notification contact for your cluster.

## Networking

This section provides information about the service definition for Azure Red Hat OpenShift with hosted control planes networking.

### Custom domains for applications

!!! warning
    Starting with Azure Red Hat OpenShift with hosted control planes 4.14, the Custom Domain Operator is deprecated. To manage Ingress in Azure Red Hat OpenShift with hosted control planes 4.14 or later, use the Ingress Operator. The functionality is unchanged for Azure Red Hat OpenShift with hosted control planes 4.13 and earlier versions.

To use a custom hostname for a route, you must update your DNS provider by creating a canonical name (CNAME) record. Your CNAME record should map the OpenShift canonical router hostname to your custom domain. The OpenShift canonical router hostname is shown on the *Route Details* page after a route is created. Alternatively, a wildcard CNAME record can be created once to route all subdomains for a given hostname to the cluster's router.

### Domain validated certificates

Azure Red Hat OpenShift with hosted control planes includes TLS security certificates needed for both internal and external services on the cluster. For external routes, there are two separate TLS wildcard certificates that are provided and installed on each cluster: one is for the web console and route default hostnames, and the other is for the API endpoint. OneCert is the certificate authority used for certificates. Routes within the cluster, such as the internal [API endpoint](https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster/#accessing-the-api-from-a-pod), use TLS certificates signed by the cluster's built-in certificate authority and require the CA bundle available in every pod for trusting the TLS certificate.

### Custom certificate authorities for builds

Azure Red Hat OpenShift with hosted control planes supports the use of custom certificate authorities to be trusted by builds when pulling images from an image registry.

### Load balancers

Azure Red Hat OpenShift with hosted control planes only deploys load balancers from the default ingress controller. All other load balancers can be optionally deployed by a customer for secondary ingress controllers or Service load balancers.

### Cluster ingress

Project administrators can add route annotations for many different purposes, including ingress control through IP allow-listing.

Ingress policies can also be changed by using `NetworkPolicy` objects, which leverage the `ovs-networkpolicy` plugin. This allows for full control over the ingress network policy down to the pod level, including between pods on the same cluster and even in the same namespace.

All cluster ingress traffic will go through the defined load balancers. Direct access to all nodes is blocked by cloud configuration.

### Cluster egress

Pod egress traffic control through `EgressNetworkPolicy` objects can be used to prevent or limit outbound traffic in Azure Red Hat OpenShift with hosted control planes.

### Cloud network configuration

Azure Red Hat OpenShift with hosted control planes allows for the configuration of a private network connection through Azure services, such as:

* Vnet    
* Vnet peering  
* Transit Gateway  
* Direct Connect


!!! important
    Red Hat site reliability engineers (SREs) do not monitor private network connections. Monitoring these connections is the responsibility of the customer.

### DNS forwarding

Azure Red Hat OpenShift with hosted control planes clusters that have a private cloud network configuration, a customer can specify internal DNS servers available on that private connection, that should be queried for explicitly provided domains.

### Network verification

Network verification checks run automatically when you deploy a Azure Red Hat OpenShift with hosted control planes cluster into an existing Virtual Private Cloud (VPC) or create an additional machine pool with a subnet that is new to your cluster. The checks validate your network configuration and highlight errors, enabling you to resolve configuration issues prior to deployment.

You can also run the network verification checks manually to validate the configuration for an existing cluster.

## Storage

This section provides information about the service definition for Azure Red Hat OpenShift with hosted control planes storage.

### Encrypted-at-rest OS and node storage

Worker nodes use encrypted-at-rest Amazon Elastic Block Store (Amazon EBS) storage.

### Encrypted-at-rest PV

EBS volumes that are used for PVs are encrypted-at-rest by default.

### Block storage (RWO)

Persistent volumes (PVs) are backed by Azure Disk which is Read-Write-Once.

PVs can be attached only to a single node at a time and are specific to the availability zone in which they were provisioned. However, PVs can be attached to any node in the availability zone.

Each cloud provider has its own limits for how many PVs can be attached to a single node. [See the specific instance types for documented maximums](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-types) 

### Shared Storage (RWX)

The Azure Files CSI Driver can be used to provide RWX support for Azure Red Hat OpenShift with hosted control planes. A Red Hat Operator is provided and installed to simplify usage.

## Platform

This section provides information about the service definition for the Azure Red Hat OpenShift with hosted control planes platform.

### Autoscaling

Node autoscaling is available on Azure Red Hat OpenShift with hosted control planes. You can configure the autoscaler option to automatically scale the number of machines in a cluster.

### Daemonsets

Customers can create and run daemonsets on Azure Red Hat OpenShift with hosted control planes. To restrict daemonsets to only running on worker nodes, use the following `nodeSelector`:

``` yaml
spec:
  nodeSelector:
    role: worker
```

## Node labels

Custom node labels are created by Red Hat during node creation and cannot be changed on Azure Red Hat OpenShift with hosted control planes clusters at this time. However, custom labels are supported when creating new machine pools.

## Node lifecycle

Worker nodes are not guaranteed longevity, and may be replaced at any time as part of the normal operation and management of OpenShift.

A worker node might be replaced in the following circumstances:

* Machine health checks are deployed and configured to ensure that a worker node with a `NotReady` status is replaced to ensure smooth operation of the cluster.  
* An Azure instance may be terminated when Azure detects irreparable failure of the underlying hardware that hosts the instance.  
* During upgrades, a new, upgraded node is first created and joined to the cluster. Once this new node has been successfully integrated into the cluster via the previously described automated health checks, an older node is then removed from the cluster.

For all containerized workloads running on a Kubernetes based system, it is best practice to configure applications to be resilient of node replacements.

### Cluster backup policy

Red Hat recommends object-level backup solutions for Azure Red Hat OpenShift with hosted control planes clusters. OpenShift API for Data Protection (OADP) is included in OpenShift but not enabled by default. Customers can configure OADP on their clusters to achieve object-level backup and restore capabilities.

Red Hat does not back up customer applications or application data. Customers are solely responsible for applications and their data, and must put their own backup and restore capabilities in place.

!!! warning 
    Customers are solely responsible for backing up and restoring their applications and application data. For more information about customer responsibilities, see "Shared responsibility matrix".

### OpenShift version

Azure Red Hat OpenShift with hosted control planes is run as a service and is kept up to date with the latest OpenShift Container Platform version. Upgrade scheduling to the latest version is available.

### Windows Containers

Red Hat OpenShift support for Windows Containers is not available on Azure Red Hat OpenShift with hosted control planes at this time.

### Container engine

Azure Red Hat OpenShift with hosted control planes runs on OpenShift 4 and uses [CRI-O](https://www.redhat.com/en/blog/red-hat-openshift-container-platform-4-now-defaults-cri-o-underlying-container-engine) as the only available container engine.

### Operating system

Azure Red Hat OpenShift with hosted control planes runs on OpenShift 4 and uses Red Hat CoreOS as the operating system for all control plane and worker nodes.

### Red Hat Operator support

Red Hat workloads typically refer to Red Hat-provided Operators made available through Operator Hub. Red Hat workloads are not managed by the Red Hat SRE team, and must be deployed on worker nodes. These Operators may require additional Red Hat subscriptions, and may incur additional cloud infrastructure costs. Examples of these Red Hat-provided Operators are:

* Red Hat Quay  
* Red Hat Advanced Cluster Management  
* Red Hat Advanced Cluster Security  
* Red Hat OpenShift Service Mesh  
* OpenShift Serverless  
* Red Hat OpenShift Logging  
* Red Hat OpenShift Pipelines

### Kubernetes Operator support

All Operators listed in the OperatorHub marketplace should be available for installation. These Operators are considered customer workloads, and are not monitored by Red Hat SRE.

## Security

This section provides information about the service definition for Azure Red Hat OpenShift with hosted control planes security.

### Authentication provider

Authentication for the cluster can be configured using either ARM API or Azure CLI. Azure Red Hat OpenShift with hosted control planes is not an identity provider, and all access to the cluster must be managed by the customer as part of their integrated solution. The use of multiple identity providers provisioned at the same time is supported.

### Privileged containers

Privileged containers are available for users with the `cluster-admin` role. Usage of privileged containers as `cluster-admin` is subject to the responsibilities and exclusion notes in the [Red Hat Enterprise Agreement Appendix 4](https://www.redhat.com/en/about/agreements) (Online Subscription Services).

### Cluster administration role

The administrator of Azure Red Hat OpenShift with hosted control planes has default access to the `cluster-admin` role for your organization’s cluster. While logged into an account with the `cluster-admin` role, users have increased permissions to run privileged security contexts.

### Project self-service

By default, all users have the ability to create, update, and delete their projects. This can be restricted if a member of the `dedicated-admin` group removes the `self-provisioner` role from authenticated users:

``` console linenums="0"
$ oc adm policy remove-cluster-role-from-group self-provisioner system:authenticated:oauth
```

Restrictions can be reverted by applying:

``` console linenums="0"
$ oc adm policy add-cluster-role-to-group self-provisioner system:authenticated:oauth
```

### Network security

With Azure Red Hat OpenShift with hosted control planes, Azure provides a standard DDoS protection on all load balancers, called Azure DDoS. This provides 95% protection against most commonly used level 3 and 4 attacks on all the public facing load balancers used for Azure Red Hat OpenShift with hosted control planes. A 10-second timeout is added for HTTP requests coming to the `haproxy` router to receive a response or the connection is closed to provide additional protection.

## Virtual machine sizes

For a list of available worker node virtual machine types and sizes, see [Worker nodes](https://learn.microsoft.com/en-us/azure/openshift/support-policies-v4#worker-nodes).

!!! note
    Microsoft Azure Red Hat OpenShift with hosted control planes supports a maximum of 500 worker nodes. |
