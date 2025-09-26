
# Create a cluster

Microsoft Azure Red Hat OpenShift with hosted control planes is a managed OpenShift service that lets you quickly deploy and manage clusters. With Azure Red Hat OpenShift with hosted control planes, each cluster has a dedicated control plane that is isolated in an Azure service account.

This article shows you how to deploy an Azure Red Hat OpenShift with hosted control planes cluster using a Bicep file.

## Prerequisites

Before deploying an Microsoft Azure Red Hat OpenShift with hosted control planes cluster, verify each of the following prerequisites.

### Azure CLI requirement

Ensure you're using Azure CLI version 2.67.0 or higher. Use `az --version` to find the version of Azure CLI you installed. If you need to install or upgrade, see [Install Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli).

### Verify resource quota

Ensure you have enough resource quota. Azure Red Hat OpenShift with hosted control planes requires a minimum of 20 cores to create and run an OpenShift cluster. The default Azure resource quota for a new Azure subscription doesn't meet the minimum cores requirement. To request an increase in your resource limit, see [Standard quota: Increase limits by VM series](https://learn.microsoft.com/en-us/azure/azure-portal/supportability/per-vm-quota-requests).

For example, to check the current subscription quota of the smallest supported virtual machine family SKU "Standard DSv5":

```
$ LOCATION=eastus
$ az vm list-usage -l $LOCATION \
  --query "[?contains(name.value, 'standardDSv5Family')]" -o table
```

### Verify permissions

In this article, you create a resource group which contains the virtual network and managed identities for the cluster. To create a resource group, you need Contributor and User Access Administrator permissions or Owner permissions on the resource group or subscription containing it. For more information, see [Verify your permissions](https://learn.microsoft.com/en-us/azure/openshift/howto-create-openshift-cluster?tabs=bicep#verify-your-permissions).

### Register resource providers

The following resource providers must be registered in your Azure subscription:

* `Microsoft.RedHatOpenShift`  
* `Microsoft.Compute`  
* `Microsoft.Storage`  
* `Microsoft.Authorization`

For more information about how to register any of these resource providers, see [Register the resource providers](https://learn.microsoft.com/en-us/azure/openshift/create-cluster?tabs=azure-cli#register-the-resource-providers).

## Create a Bicep file

Create a Bicep file that defines an Azure Red Hat OpenShift with hosted control planes cluster.

### Default resources to define

To create an Azure Red Hat OpenShift with hosted control planes cluster, you must define the following resources:

* Network Security Group (NSG)  
* Virtual network with an empty subnet  
* A user-assigned managed identity and role assignment for each OpenShift cluster Operator  
* A service managed identity and role assignments  
* Hosted control plane (`Microsoft.RedHatOpenShift/hcpOpenShiftClusters` resource type)  
* Node pool (`Microsoft.RedHatOpenShift/hcpOpenShiftClusters/nodePools` resource type)

### Resources to define for customer-managed etcd encryption

If you want to use your own key to encrypt etcd, you must also define these additional resources:

* Azure Key Vault
* Custom KMS key to encrypt the etcd database
* A user-assigned managed identity to access the KMS key, and a role assignment that grants Key Vault permissions
* A role assignment that grants `Reader` permissions to the service managed identity 

### Example Bicep file

This example Bicep file defines a cluster that uses a custom KMS key to encrypt the etcd database. You can customize the cluster and node pool configuration properties as needed.

**Example `azuredeploy.bicep` file**
```
@description('Network Security Group Name')
param customerNsgName string

@description('Virtual Network Name')
param customerVnetName string

@description('Subnet Name')
param customerVnetSubnetName string

@description('Name of the hypershift cluster')
param clusterName string

@description('The Hypershift cluster managed resource group name')
param managedResourceGroupName string

@description('The name of the node pool')
param nodePoolName string

var randomSuffix = toLower(uniqueString(clusterName))
var addressPrefix = '10.0.0.0/16'
var subnetPrefix = '10.0.0.0/24'

resource customerNsg 'Microsoft.Network/networkSecurityGroups@2023-05-01' = {
  name: customerNsgName
  location: resourceGroup().location
  tags: {
    persist: 'true'
  }
}

resource customerVnet 'Microsoft.Network/virtualNetworks@2023-05-01' = {
  name: customerVnetName
  location: resourceGroup().location
  tags: {
    persist: 'true'
  }
  properties: {
    addressSpace: {
      addressPrefixes: [
        addressPrefix
      ]
    }
    subnets: [
      {
        name: customerVnetSubnetName
        properties: {
          addressPrefix: subnetPrefix
          networkSecurityGroup: {
            id: customerNsg.id
          }
        }
      }
    ]
  }
}

resource subnet 'Microsoft.Network/virtualNetworks/subnets@2022-07-01' existing = {
  name: customerVnetSubnetName
  parent: customerVnet
}

//
// C O N T R O L   P L A N E   I D E N T I T I E S
//

// Reader
var readerRoleId = subscriptionResourceId(
  'Microsoft.Authorization/roleDefinitions',
  'acdd72a7-3385-48ef-bd42-f606fba81ae7'
)

//
// C L U S T E R   A P I   A Z U R E   M I
//

resource clusterApiAzureMi 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: '${clusterName}-cp-cluster-api-azure-${randomSuffix}'
  location: resourceGroup().location
}

// Azure Red Hat OpenShift Hosted Control Planes Cluster API Provider
var hcpClusterApiProviderRoleId = subscriptionResourceId(
  'Microsoft.Authorization/roleDefinitions',
  '88366f10-ed47-4cc0-9fab-c8a06148393e'
)

resource hcpClusterApiProviderRoleSubnetAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, clusterApiAzureMi.id, hcpClusterApiProviderRoleId, subnet.id)
  scope: subnet
  properties: {
    principalId: clusterApiAzureMi.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: hcpClusterApiProviderRoleId
  }
}

resource serviceManagedIdentityReaderOnClusterApiAzureMi 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, serviceManagedIdentity.id, readerRoleId, clusterApiAzureMi.id)
  scope: clusterApiAzureMi
  properties: {
    principalId: serviceManagedIdentity.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: readerRoleId
  }
}

//
// C O N T R O L   P L A N E   O P E R A T O R   M A N A G E D   I D E N T I T Y
//

resource controlPlaneMi 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: '${clusterName}-cp-control-plane-${randomSuffix}'
  location: resourceGroup().location
}

// Azure Red Hat OpenShift Hosted Control Planes Control Plane Operator
var hcpControlPlaneOperatorRoleId = subscriptionResourceId(
  'Microsoft.Authorization/roleDefinitions',
  'fc0c873f-45e9-4d0d-a7d1-585aab30c6ed'
)

resource hcpControlPlaneOperatorVnetRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, controlPlaneMi.id, hcpControlPlaneOperatorRoleId, customerVnet.id)
  scope: customerVnet
  properties: {
    principalId: controlPlaneMi.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: hcpControlPlaneOperatorRoleId
  }
}

resource hcpControlPlaneOperatorNsgRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, controlPlaneMi.id, hcpControlPlaneOperatorRoleId, customerNsg.id)
  scope: customerNsg
  properties: {
    principalId: controlPlaneMi.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: hcpControlPlaneOperatorRoleId
  }
}

resource serviceManagedIdentityReaderOnControlPlaneMi 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, serviceManagedIdentity.id, readerRoleId, controlPlaneMi.id)
  scope: controlPlaneMi
  properties: {
    principalId: serviceManagedIdentity.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: readerRoleId
  }
}

//
// C L O U D   C O N T R O L L E R   M A N A G E R   M A N A G E D   I D E N T I T Y
//

resource cloudControllerManagerMi 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: '${clusterName}-cp-cloud-controller-manager-${randomSuffix}'
  location: resourceGroup().location
}

// Azure Red Hat OpenShift Cloud Controller Manager
var cloudControllerManagerRoleId = subscriptionResourceId(
  'Microsoft.Authorization/roleDefinitions',
  'a1f96423-95ce-4224-ab27-4e3dc72facd4'
)

resource cloudControllerManagerRoleSubnetAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, cloudControllerManagerMi.id, cloudControllerManagerRoleId, subnet.id)
  scope: subnet
  properties: {
    principalId: cloudControllerManagerMi.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: cloudControllerManagerRoleId
  }
}

resource cloudControllerManagerRoleNsgAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, cloudControllerManagerMi.id, cloudControllerManagerRoleId, customerNsg.id)
  scope: customerNsg
  properties: {
    principalId: cloudControllerManagerMi.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: cloudControllerManagerRoleId
  }
}

resource serviceManagedIdentityReaderOnCloudControllerManagerMi 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, serviceManagedIdentity.id, readerRoleId, cloudControllerManagerMi.id)
  scope: cloudControllerManagerMi
  properties: {
    principalId: serviceManagedIdentity.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: readerRoleId
  }
}

//
// I N G R E S S   M A N A G E D   I D E N T I T Y
//

resource ingressMi 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: '${clusterName}-cp-ingress-${randomSuffix}'
  location: resourceGroup().location
}

// Azure Red Hat OpenShift Cluster Ingress Operator
var ingressOperatorRoleId = subscriptionResourceId(
  'Microsoft.Authorization/roleDefinitions',
  '0336e1d3-7a87-462b-b6db-342b63f7802c'
)

resource ingressOperatorRoleSubnetAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, ingressMi.id, ingressOperatorRoleId, subnet.id)
  scope: subnet
  properties: {
    principalId: ingressMi.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: ingressOperatorRoleId
  }
}

resource serviceManagedIdentityReaderOnIngressMi 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, serviceManagedIdentity.id, readerRoleId, ingressMi.id)
  scope: ingressMi
  properties: {
    principalId: serviceManagedIdentity.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: readerRoleId
  }
}

//
// D I S K   C S I   D R I V E R   M A N A G E D   I D E N T I T Y
//

resource diskCsiDriverMi 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: '${clusterName}-cp-disk-csi-driver-${randomSuffix}'
  location: resourceGroup().location
}

resource serviceManagedIdentityReaderOnDiskCsiDriverMi 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, serviceManagedIdentity.id, readerRoleId, diskCsiDriverMi.id)
  scope: diskCsiDriverMi
  properties: {
    principalId: serviceManagedIdentity.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: readerRoleId
  }
}

//
// F I L E   C S I   D R I V E R   M A N A G E D   I D E N T I T Y
//

resource fileCsiDriverMi 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: '${clusterName}-cp-file-csi-driver-${randomSuffix}'
  location: resourceGroup().location
}

// Azure Red Hat OpenShift File Storage Operator
var fileStorageOperatorRoleId = subscriptionResourceId(
  'Microsoft.Authorization/roleDefinitions',
  '0d7aedc0-15fd-4a67-a412-efad370c947e'
)

resource fileStorageOperatorRoleSubnetAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, fileCsiDriverMi.id, fileStorageOperatorRoleId, subnet.id)
  scope: subnet
  properties: {
    principalId: fileCsiDriverMi.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: fileStorageOperatorRoleId
  }
}

resource fileStorageOperatorRoleNsgAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, fileCsiDriverMi.id, fileStorageOperatorRoleId, customerNsg.id)
  scope: customerNsg
  properties: {
    principalId: fileCsiDriverMi.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: fileStorageOperatorRoleId
  }
}

resource serviceManagedIdentityReaderOnFileCsiDriverMi 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, serviceManagedIdentity.id, readerRoleId, fileCsiDriverMi.id)
  scope: fileCsiDriverMi
  properties: {
    principalId: serviceManagedIdentity.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: readerRoleId
  }
}

//
// I M A G E   R E G I S T R Y   M A N A G E D   I D E N T I T Y
//

resource imageRegistryMi 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: '${clusterName}-cp-image-registry-${randomSuffix}'
  location: resourceGroup().location
}

resource serviceManagedIdentityReaderOnImageRegistryMi 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, serviceManagedIdentity.id, readerRoleId, imageRegistryMi.id)
  scope: imageRegistryMi
  properties: {
    principalId: serviceManagedIdentity.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: readerRoleId
  }
}

//
// C L O U D   N E T W O R K   C O N F I G   M A N A G E D   I D E N T I T Y
//

resource cloudNetworkConfigMi 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: '${clusterName}-cp-cloud-network-config-${randomSuffix}'
  location: resourceGroup().location
}

// Azure Red Hat OpenShift Network Operator
var networkOperatorRoleId = subscriptionResourceId(
  'Microsoft.Authorization/roleDefinitions',
  'be7a6435-15ae-4171-8f30-4a343eff9e8f'
)

resource networkOperatorRoleSubnetAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, cloudNetworkConfigMi.id, networkOperatorRoleId, subnet.id)
  scope: subnet
  properties: {
    principalId: cloudNetworkConfigMi.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: networkOperatorRoleId
  }
}

resource networkOperatorRoleVnetAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, cloudNetworkConfigMi.id, networkOperatorRoleId, customerVnet.id)
  scope: customerVnet
  properties: {
    principalId: cloudNetworkConfigMi.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: networkOperatorRoleId
  }
}

resource serviceManagedIdentityReaderOnCloudNetworkMi 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, serviceManagedIdentity.id, readerRoleId, cloudNetworkConfigMi.id)
  scope: cloudNetworkConfigMi
  properties: {
    principalId: serviceManagedIdentity.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: readerRoleId
  }
}

//
// D A T A P L A N E   I D E N T I T I E S
//

// Azure Red Hat OpenShift Federated Credential
// give the ability to perform OIDC federation to the service managed identity
// over the corresponding dataplane identities
var federatedCredentialsRoleId = subscriptionResourceId(
  'Microsoft.Authorization/roleDefinitions',
  'ef318e2a-8334-4a05-9e4a-295a196c6a6e'
)

resource dpDiskCsiDriverMi 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: '${clusterName}-dp-disk-csi-driver-${randomSuffix}'
  location: resourceGroup().location
}

resource dpDiskCsiDriverMiFederatedCredentialsRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, dpDiskCsiDriverMi.id, federatedCredentialsRoleId)
  scope: dpDiskCsiDriverMi
  properties: {
    principalId: serviceManagedIdentity.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: federatedCredentialsRoleId
  }
}

resource dpFileCsiDriverMi 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: '${clusterName}-dp-file-csi-driver-${randomSuffix}'
  location: resourceGroup().location
}

resource dpFileCsiDriverMiFederatedCredentialsRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, dpFileCsiDriverMi.id, federatedCredentialsRoleId)
  scope: dpFileCsiDriverMi
  properties: {
    principalId: serviceManagedIdentity.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: federatedCredentialsRoleId
  }
}

resource dpFileCsiDriverFileStorageOperatorRoleSubnetAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, dpFileCsiDriverMi.id, fileStorageOperatorRoleId, subnet.id)
  scope: subnet
  properties: {
    principalId: dpFileCsiDriverMi.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: fileStorageOperatorRoleId
  }
}

resource dpFileCsiDriverFileStorageOperatorRoleNsgAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, dpFileCsiDriverMi.id, fileStorageOperatorRoleId, customerNsg.id)
  scope: customerNsg
  properties: {
    principalId: dpFileCsiDriverMi.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: fileStorageOperatorRoleId
  }
}

resource dpImageRegistryMi 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: '${clusterName}-dp-image-registry-${randomSuffix}'
  location: resourceGroup().location
}

resource dpImageRegistryMiFederatedCredentialsRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, dpImageRegistryMi.id, federatedCredentialsRoleId)
  scope: dpImageRegistryMi
  properties: {
    principalId: serviceManagedIdentity.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: federatedCredentialsRoleId
  }
}

//
// S E R V I C E   M A N A G E D   I D E N T I T Y
//

resource serviceManagedIdentity 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: '${clusterName}-service-managed-identity-${randomSuffix}'
  location: resourceGroup().location
}

// Azure Red Hat OpenShift Hosted Control Planes Service Managed Identity
var hcpServiceManagedIdentityRoleId = subscriptionResourceId(
  'Microsoft.Authorization/roleDefinitions',
  'c0ff367d-66d8-445e-917c-583feb0ef0d4'
)

// grant service managed identity role to the service managed identity over the user provided subnet
resource serviceManagedIdentityRoleAssignmentVnet 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, serviceManagedIdentity.id, hcpServiceManagedIdentityRoleId, customerVnet.id)
  scope: customerVnet
  properties: {
    principalId: serviceManagedIdentity.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: hcpServiceManagedIdentityRoleId
  }
}

// grant service managed identity role to the service managed identity over the user provided subnet
resource serviceManagedIdentityRoleAssignmentSubnet 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, serviceManagedIdentity.id, hcpServiceManagedIdentityRoleId, subnet.id)
  scope: subnet
  properties: {
    principalId: serviceManagedIdentity.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: hcpServiceManagedIdentityRoleId
  }
}

// grant service managed identity role to the service managed identity over the user provided NSG
resource serviceManagedIdentityRoleAssignmentNSG 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, serviceManagedIdentity.id, hcpServiceManagedIdentityRoleId, customerNsg.id)
  scope: customerNsg
  properties: {
    principalId: serviceManagedIdentity.properties.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: hcpServiceManagedIdentityRoleId
  }
}

resource hcp 'Microsoft.RedHatOpenShift/hcpOpenShiftClusters@2024-06-10-preview' = {
  name: clusterName
  location: resourceGroup().location
  properties: {
    version: {
      id: '4.19'
      channelGroup: 'stable'
    }
    dns: {}
    network: {
      networkType: 'OVNKubernetes'
      podCidr: '10.128.0.0/14'
      serviceCidr: '172.30.0.0/16'
      machineCidr: '10.0.0.0/16'
      hostPrefix: 23
    }
    console: {}
    etcd: {
      dataEncryption: {
        keyManagementMode: 'PlatformManaged'
      }
    }
    api: {
      visibility: 'Public'
    }
    clusterImageRegistry: {
      state: 'Enabled'
    }
    platform: {
      managedResourceGroup: managedResourceGroupName
      subnetId: subnet.id
      outboundType: 'LoadBalancer'
      networkSecurityGroupId: customerNsg.id
      operatorsAuthentication: {
        userAssignedIdentities: {
          controlPlaneOperators: {
            'cluster-api-azure': clusterApiAzureMi.id
            'control-plane': controlPlaneMi.id
            'cloud-controller-manager': cloudControllerManagerMi.id
            #disable-next-line prefer-unquoted-property-names
            'ingress': ingressMi.id
            'disk-csi-driver': diskCsiDriverMi.id
            'file-csi-driver': fileCsiDriverMi.id
            'image-registry': imageRegistryMi.id
            'cloud-network-config': cloudNetworkConfigMi.id
          }
          dataPlaneOperators: {
            'disk-csi-driver': dpDiskCsiDriverMi.id
            'file-csi-driver': dpFileCsiDriverMi.id
            'image-registry': dpImageRegistryMi.id
          }
          serviceManagedIdentity: serviceManagedIdentity.id
        }
      }
    }
  }
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: {
      '${serviceManagedIdentity.id}': {}
      '${clusterApiAzureMi.id}': {}
      '${controlPlaneMi.id}': {}
      '${cloudControllerManagerMi.id}': {}
      '${ingressMi.id}': {}
      '${diskCsiDriverMi.id}': {}
      '${fileCsiDriverMi.id}': {}
      '${imageRegistryMi.id}': {}
      '${cloudNetworkConfigMi.id}': {}
    }
  }
  dependsOn: [
    hcpClusterApiProviderRoleSubnetAssignment
    hcpControlPlaneOperatorVnetRoleAssignment
    hcpControlPlaneOperatorNsgRoleAssignment
    cloudControllerManagerRoleSubnetAssignment
    cloudControllerManagerRoleNsgAssignment
    ingressOperatorRoleSubnetAssignment
    fileStorageOperatorRoleSubnetAssignment
    fileStorageOperatorRoleNsgAssignment
    networkOperatorRoleSubnetAssignment
    networkOperatorRoleVnetAssignment
    dpDiskCsiDriverMiFederatedCredentialsRoleAssignment
    dpFileCsiDriverMiFederatedCredentialsRoleAssignment
    dpImageRegistryMiFederatedCredentialsRoleAssignment
    serviceManagedIdentityRoleAssignmentVnet
    serviceManagedIdentityRoleAssignmentSubnet
    serviceManagedIdentityRoleAssignmentNSG
    dpFileCsiDriverFileStorageOperatorRoleSubnetAssignment
    dpFileCsiDriverFileStorageOperatorRoleNsgAssignment
    serviceManagedIdentityReaderOnControlPlaneMi
    serviceManagedIdentityReaderOnCloudControllerManagerMi
    serviceManagedIdentityReaderOnIngressMi
    serviceManagedIdentityReaderOnDiskCsiDriverMi
    serviceManagedIdentityReaderOnFileCsiDriverMi
    serviceManagedIdentityReaderOnImageRegistryMi
    serviceManagedIdentityReaderOnCloudNetworkMi
    serviceManagedIdentityReaderOnClusterApiAzureMi
  ]
}

resource nodepool 'Microsoft.RedHatOpenShift/hcpOpenShiftClusters/nodePools@2024-06-10-preview' = {
  parent: hcp
  name: nodePoolName
  location: resourceGroup().location
  properties: {
    version: {
      id: '4.19.0'
      channelGroup: 'stable'
    }
    platform: {
      subnetId: hcp.properties.platform.subnetId
      vmSize: 'Standard_D8s_v3'
      osDisk: {
        sizeGiB: 64
        diskStorageAccountType: 'StandardSSD_LRS'
      }
    }
    replicas: 2
  }
}	
```

## Create the cluster using Bicep

This section describes how to use Bicep to create the cluster. 

1. Set the following variables in the shell environment that you plan to execute the az commands.

   ```
   # location of your cluster
   LOCATION="<location>"

   # name of your subscription
   SUBSCRIPTION="<subscription-name>"

   # name of the resource group where you want to create your cluster
   CUSTOMER_RG_NAME="$USER-net-rg-03"

   # name of the nsg
   CUSTOMER_NSG="<nsg-name>"

   # name of the vnet
   CUSTOMER_VNET_NAME="<vnet-name>"

   # name of the subnet within the vnet
   CUSTOMER_VNET_SUBNET1="<subnet-name>"

   # name of your cluster
   CLUSTER_NAME="<cluster-name>"

   # name of the managed resource group
   MANAGED_RESOURCE_GROUP="$CLUSTER_NAME-rg-03"

   # name of the node pool
   NP_NAME="<node-pool-name>"

   # optional variables for creating tags
   SUBSCRIPTION_ID=$(az account show --query id --output tsv)
   POLICY_DEFINITION="<policy-name>"
   POLICY_ASSIGNMENT="${POLICY_DEFINITION}-${CLUSTER_NAME}"
   ```

1. Create a resource group to hold the cluster resource, node pool resource, and the cluster virtual network and identities.

   ```
   $ az group create \
     --name "${CUSTOMER_RG_NAME}" \
     --subscription "${SUBSCRIPTION}" \
     --location "${LOCATION}"
   ```

1. (Optional) If desired, define tags that will be applied to each of the resources in your managed resource group.

    Follow these steps to use Azure Policy to tag the resources in your managed resource group. When you create the Azure Red Hat OpenShift with hosted control planes cluster, these tags will be applied to the managed resource group, and to each of the resources within it:

     1. Create the following JSON files to define the tags:

         * [Rules file](https://learn.microsoft.com/en-us/azure/openshift/howto-tag-resources#create-rules-file)
         * [Parameter definitions](https://learn.microsoft.com/en-us/azure/openshift/howto-tag-resources#create-parameter-definitions)
         * [Parameter values](https://learn.microsoft.com/en-us/azure/openshift/howto-tag-resources#create-parameter-definitions)
    
     1. Create the policy definition.

        ```
        $ az policy definition create -n $POLICY_DEFINITION \
          --mode All \
          --rules rules.json \
          --params param-defs.json
        ```

     1. Create the policy assignment.

        ```
        $ az policy assignment create -n $POLICY_ASSIGNMENT \
          --policy $POLICY_DEFINITION \
          --scope "/subscriptions/${SUBSCRIPTION_ID}" \
          --location $LOCATION \
          --mi-system-assigned \
          --role "Tag Contributor" \
          --identity-scope "/subscriptions/${SUBSCRIPTION_ID}" \
          --params param-values.json
        ```

1. Apply the Bicep file.

    ```
    $ az deployment group create \
    --name 'aro-hcp' \
    --subscription "${SUBSCRIPTION}" \
    --resource-group "${CUSTOMER_RG_NAME}" \
    --template-file azuredeploy.bicep \
    --parameters \
      customerNsgName="${CUSTOMER_NSG}" \
      customerVnetName="${CUSTOMER_VNET_NAME}" \
      customerVnetSubnetName="${CUSTOMER_VNET_SUBNET1}" \
      clusterName="${CLUSTER_NAME}" \
      managedResourceGroupName="${MANAGED_RESOURCE_GROUP}" \
      nodePoolName="${NP_NAME}"
    ```

1. (Optional) If desired, create a tag for the managed resource group.

    ```
    $ az tag create \
    --resource-id /subscriptions/${SUBSCRIPTION_ID}/resourcegroups/${MANAGED_RESOURCE_GROUP} \
    --tags <tag-key>=<tag-value>
    ```

## Access the cluster

After creating an Azure Red Hat OpenShift with hosted control planes cluster, you can request a temporary administrative credential to access your cluster.

Requesting an administrative credential generates a new cluster-admin `kubeconfig` file. The `kubeconfig` file contains information about the cluster that connects a client to the correct cluster and API server. You can use the newly generated `kubeconfig` file to allow access to the cluster.

Administrative credentials expire after 24 hours. You can request multiple administrative credentials for a cluster. You can also revoke all administrative credentials for the cluster.

1. Request an administrative credential.

    ```
    $ az rest \
      --method POST \
      --uri "/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${CUSTOMER_RG_NAME}/providers/Microsoft.RedHatOpenShift/hcpOpenShiftClusters/${CLUSTER_NAME}/requestAdminCredential?api-version=2024-06-10-preview" \
      --verbose
    ```

	The admin credential is generated as an [asynchronous Azure operation](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/async-operations).  

1. Monitor the status of the operation.

    1. In the response headers, find the URL listed in the `Azure-AsyncOperation` field.

    1. Use the `Azure-AsyncOperation` URL to get the status.

        ```
        $ az rest \
          --method GET \
          --uri "<url-from-Azure-AsyncOperation-field>"
        ```

1. Get the newly-generated `kubeconfig` content and save it as a `kubeconfig` file.

    1. In the response headers from the original operation, find the URL listed in the `Location` field.

    1. Use the `Location` field URL to get the `kubeconfig` file.

        ```
        $ az rest \
          --method GET \
          --uri "<url-from-Location-field>" \
          |  jq -r '.kubeconfig' > kubeconfig
        ```

1. Verify that you have the correct credentials:

    ```
    $ export KUBECONFIG=kubeconfig
    $ kubectl auth whoami --insecure-skip-tls-verify=true
    ATTRIBUTE    VALUE
    Username     system:customer-break-glass:<user-id>
    Groups       [system:masters system:authenticated]
    Extra: authentication.kubernetes.io/credential-id   <credential-id>
    ```

1. If desired, revoke all administrative credentials for the cluster:

    ```
    $ az rest \
      --method POST \
      --uri "/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${CUSTOMER_RG_NAME}/providers/Microsoft.RedHatOpenShift/hcpOpenShiftClusters/${CLUSTER_NAME}/revokeCredentials?api-version=2024-06-10-preview"
    ```

## Clean up

When you are done using your Azure Red Hat OpenShift with hosted control planes cluster, you can delete it.

1. Delete the cluster (the `hcpOpenShiftClusters` resource).

    Deleting the `hcpOpenShiftClusters` resource deletes the cluster, its node pools, and the managed resource group.

    ```
    $ az rest \
      --method DELETE \
      --uri "/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${CUSTOMER_RG_NAME}/providers/Microsoft.RedHatOpenShift/hcpOpenShiftClusters/${CLUSTER_NAME}?api-version=2024-06-10-preview"
    ```

1. Verify that the cluster is deleted.

    ```
    $ az rest \
      --method GET \
      --uri "/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${CUSTOMER_RG_NAME}/providers/Microsoft.RedHatOpenShift/hcpOpenShiftClusters/${CLUSTER_NAME}?api-version=2024-06-10-preview"
    ```

1. Delete the resource group.

    ```
    $ az group delete --name ${CUSTOMER_RG_NAME}
    ```
