
# Create an external authentication provider

In OpenShift, an external authentication provider is any system outside the cluster that handles user authentication on its behalf. Instead of creating and managing local accounts within OpenShift, you can connect the cluster to Microsoft Entra ID.

This article describes how to configure Microsoft Entra ID as an external authentication provider to provide access to both the CLI and the OpenShift console.

## Prepare environment variables

Make sure that the following environment variables are defined:

```
SUBSCRIPTION="<subscription-name>"
SUBSCRIPTION_ID=$(az account show --query id --output tsv)
CUSTOMER_RG_NAME="<resource-group-name>"
CLUSTER_NAME="<cluster-name>"
EXTERNAL_AUTH_NAME=${CLUSTER_NAME}-auth
TENANT_ID=$(az account show --query tenantId --output tsv)
ISSUER_URL="https://login.microsoftonline.com/${TENANT_ID}/v2.0"
```

## Configure Microsoft Entra ID

1. Retrieve the OpenShift Authentication call back URL.

    ``` console linenums="0"
    $ OAUTH_CALLBACK_URL=$(az rest --method GET --uri "/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${CUSTOMER_RG_NAME}/providers/Microsoft.RedHatOpenShift/hcpOpenShiftClusters/${CLUSTER_NAME}?api-version=2024-06-10-preview" | jq -r '.properties.console.url')/auth/callback
    ```

1. Register a new application in Microsoft Entra ID.

    If you would like to access the API server via the `oc` or `kubectl` CLIs, you must add a second redirect URI.

    The following example creates a Microsoft Entra ID application with two redirect URIs: the required OpenShift Authentication call back URL, and a second redirect for CLI access:

    ``` console linenums="0"
    $ CLIENT_ID=$(az ad app create \
      --display-name ${EXTERNAL_AUTH_NAME} \
      --web-redirect-uris ${OAUTH_CALLBACK_URL} http://localhost:8000 \
      --query appId --output tsv)
    ```

1. Create a client secret for the application.

    A console message is displayed notifying you that the command output contains credentials. The client secret is used later in this article. 

    !!! note
        When you’re finished, you can clear the variable by entering the `clientSecret=""` command.

    ``` console linenums="0"
    $ CLIENT_SECRET=$(az ad app credential reset --id ${CLIENT_ID} --query password --output tsv)
    ```

## Configure optional claims

To provide OpenShift with enough information to identify the user, configure Microsoft Entra ID to provide an email optional claim when a user logs in. For more information on optional claims, see [Configure and manage optional claims in ID tokens, access tokens, and SAML tokens](https://learn.microsoft.com/en-us/entra/identity-platform/optional-claims?tabs=appui).

1. Create a `manifest.json` file to configure the optional claims.

    The following example configures three optional claims: email, groups, and preferred username:

    ``` console linenums="0"
    $ cat > manifest.json<< EOF
    {
    "idToken": [
        {
        "name": "email",
        "source": null,
        "essential": false,
        "additionalProperties": []
        },
        {
        "name": "groups",
        "source": null,
        "essential": false,
        "additionalProperties": []
        }
    ],
    "accessToken": [
        {
        "name": "groups",
        "source": null,
        "essential": false,
        "additionalProperties": []
        }
    ],
    "saml2Token": [
        {
        "name": "groups",
        "source": null,
        "essential": false,
        "additionalProperties": []
        }
    ]
    }
    EOF
    ```

1. Update the Microsoft Entra ID application to use the optional claims that you defined in the manifest.

    ```
    $ az ad app update \
      --id ${CLIENT_ID} \
      --optional-claims @manifest.json
    ```

1. Assign the Microsoft Graph [email](https://learn.microsoft.com/en-us/graph/permissions-reference#email) permission to your Microsoft Entra ID application. 

    This permission enables the ability to read a user’s email address from Microsoft Entra ID.

    ``` console linenums="0"
    $ az ad app permission add \
      --api 00000003-0000-0000-c000-000000000000 \
      --api-permissions 64a6cdd6-aab1-4aaf-94b8-3cc8405e90d0=Scope \
      --id ${CLIENT_ID}
    ```

    Ignore the message to grant the consent unless you’re authenticated as a Global Administrator for this Microsoft Entra ID. Standard domain users are asked to grant consent when they first sign in to the cluster with their Microsoft Entra credentials.

## Create an external authentication provider

1. Retrieve the API URL.

    ``` console linenums="0"
    $ API_URL=$(az rest --method GET --uri "/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${CUSTOMER_RG_NAME}/providers/Microsoft.RedHatOpenShift/hcpOpenShiftClusters/${CLUSTER_NAME}?api-version=2024-06-10-preview" | jq -r '.properties.api.url')
    ```

1. Configure the external authentication provider for Azure Red Hat OpenShift with hosted control planes.

    To configure an external authentication provider, you must define an `externalAuths` resource. The following example shows an `externalAuths` resource in a Bicep file.

    !!! example "Example external authentication provider configuration (`externalauth.bicep`)"

        ```
        @description('The name of the external auth provider configuration')    
        param externalAuthName string

        @description('The issuer url')
        param issuerURL string

        @description('The client ID')
        param clientID string

        @description('Name of the hypershift cluster')
        param clusterName string

        resource hcp 'Microsoft.RedHatOpenShift/hcpOpenShiftClusters@2024-06-10-preview' existing = {
            name: clusterName
        }

        resource externalauth 'Microsoft.RedHatOpenShift/hcpOpenShiftClusters/externalAuths@2024-06-10-preview' = {
            parent: hcp
            name: externalAuthName
            properties: {
                claim: {
                    mappings: {
                        username: {
                            claim: 'email'
                        }
                        groups: {
                            claim: 'groups'
                        }
                    }
                }
                clients: [
                    {
                        clientId: clientID
                        component: {
                            name: 'console'
                            authClientNamespace: 'openshift-console'
                        }
                        type: 'confidential'
                    }
                    {
                        clientId: clientID
                        component: {
                            name: 'cli'
                            authClientNamespace: 'openshift-console'
                        }
                        type: 'public'
                    }
                ]
                issuer: {
                    url: issuerURL
                    audiences: [
                        clientID
                    ]
                }
            }
        }
        ```

        `claim.mappings.username`
        :   The name of the claim that is used to construct the user names for cluster identity. You can configure the following properties for the claim user name:

            `claim.mappings.username.claim`
            :   The claim name.

            `claim.mappings.username.prefix`
            :   Optional. The prefix for the claim external profile. If you set the prefix, `prefixPolicy` will be set to `Prefix` by default.

            `claim.mappings.username.prefixPolicy`
            :   Optional. Defines how a prefix should be applied to the value of the JWT claim specified in the claim property. You can set this to `Prefix` (the claim name will be prefixed with the value of the prefix property), `NoPrefix` (no prefix will be added to the claim name), or leave the property unset (the platform will define the claim name prefix).
        
        `claim.mappings.groups`
        :   Optional. The method with which to transform the ID token into a cluster identity. You can configure the following properties for the claim mapping groups:
        
             * `claim.mappings.groups.claim` - The claim name.
             * `claim.mappings.groups.prefix` - Optional. The prefix for the claim external profile.
        
        `claim.validationRules`
        :   Optional. The rules that help validate token claims which authenticate your users. To add a validation rule, configure the `claim.validationRules.requiredClaim` property. 

        `clients`
        :   Optional. Configures how on-cluster, platform clients should request tokens from the identity provider. Must not exceed 20 entries and entries must have unique namespace/name pairs.
        
             For each client, you can configure the following properties:
         
             * `clients.component.name`  - The name of the platform component being configured to use the identity provider as an authentication mode.
             * `clients.component.authClientNamespace` - The namespace in which the platform component being configured to use the identity provider as an authentication mode is running.
             * `clients.clientId` - The client identifier, from the identity provider, that the platform component uses for authentication requests made to the identity provider. The identity provider must accept this identifier for platform components to be able to use the identity provider as an authentication mode. The client ID must be available in the issuer audiences.
             * `clients.extraScopes` - Optional. The extra scopes that should be requested by the platform component when making authentication requests to the identity provider. This is useful if you have configured claim mappings that require specific scopes to be requested beyond the standard OIDC scopes. When omitted, no additional scopes are requested.
             * `clients.type` - The client type (either confidential or public). Confidential clients must provide a client secret for authentication requests. Public clients don’t have to provide a client secret. The only supported confidential client is `console/openshift-console`.
        
        `issuer.url`
        :   Required. The URL of the token issuer.
        
            The Kubernetes API server determines how authentication tokens should be handled by matching the `iss` claim in the JWT to the issuer URL of configured identity providers. This field must use the `https://` scheme.

        `issuer.audiences`
        :   Required. The audience IDs for which this authentication provider issues tokens. You must configure at least one audience, and can configure up to ten total audiences. At least one of the entries must match the `aud` claim in the JWT token.

        `issuer.ca`
        :   Optional. The PEM encoded certificate to use when making requests. If you don’t specify a CA, the system trust is used.

1. Apply the Bicep file.

    ``` console linenums="0"
    $ az deployment group create \
      --name 'aro-hcp-auth' \
      --subscription "${SUBSCRIPTION}" \
      --resource-group "${CUSTOMER_RG_NAME}" \
      --template-file externalauth.bicep \
      --parameters \
        externalAuthName="${EXTERNAL_AUTH_NAME}" \
        issuerURL="${ISSUER_URL}" \
        clientID="${CLIENT_ID}" \
        clusterName="${CLUSTER_NAME}"
    ```

1. Verify the external authentication provider configuration.

    ``` console linenums="0"
    $ az rest --method GET --uri "/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${CUSTOMER_RG_NAME}/providers/Microsoft.RedHatOpenShift/hcpOpenShiftClusters/${CLUSTER_NAME}/externalAuths/${EXTERNAL_AUTH_NAME}?api-version=2024-06-10-preview"
    ```

## Configure authentication and authorization in OpenShift

After creating the external authentication provider, you must log into your Azure Red Hat OpenShift with hosted control planes cluster to configure access for your confidential client (the OpenShift console), and to set up role-based access control (RBAC).

1. If you don’t have a temporary administrative credential for your Azure Red Hat OpenShift with hosted control planes cluster, generate one.

    For more information, see [Access the cluster](create-cluster.md#access-the-cluster).

    Your cluster doesn’t use the built-in OpenShift OAuth server to act as a separate identity provider. This means there isn’t a standalone cluster-admin account with which you can access the cluster. Instead, you must request a temporary administrative credential, which provides a `kubeconfig` with cluster-admin privileges.

1. Using the `kubeconfig` file from the temporary admin credential, create a secret in the `openshift-config` namespace to store the client secret for your Microsoft Entra application.

    This example creates a secret that enables the OpenShift console to use the external authentication provider for authentication:

    ``` console linenums="0"
    $ oc create secret generic ${EXTERNAL_AUTH_NAME}-console-openshift-console \
      --namespace openshift-config \
      --from-literal=clientSecret=${CLIENT_SECRET}
    ```

1. (Optional) If necessary, assign users and groups to your Azure Red Hat OpenShift with hosted control planes cluster.

    Applications registered in a Microsoft Entra tenant are, by default, available to all users of the tenant who authenticate successfully. Microsoft Entra ID allows tenant administrators and developers to restrict an app to a specific set of users or security groups in the tenant.

    Follow the instructions on the Microsoft Entra documentation to [Restrict a Microsoft Entra app to a set of users](https://learn.microsoft.com/en-us/entra/identity-platform/howto-restrict-your-app-to-a-set-of-users).

1. Configure RBAC for your cluster.

    By default, OpenShift does not grant permissions to take any action inside of your cluster when a user first logs in. Azure Red Hat OpenShift with hosted control planes includes a significant number of preconfigured roles, including the cluster-admin role that grants full access and control over the cluster. The cluster does not automatically create `RoleBinding` and `ClusterRoleBinding` objects for the groups that are included in your access token; you are responsible for creating those bindings yourself.

    This example shows how to create a `ClusterRoleBinding` object that grants members of a Microsoft Azure group the cluster-admin role.

    1. Retrieve the group ID of the group to which you want to grant access.

        ``` console linenums="0"
        $ GROUP_ID=$(az ad group show --group "<group-name>" --query id -o tsv)
        ```

    1. Create the `ClusterRoleBinding` object to grant the group access to the cluster-admin role.

        ``` console
        $ oc apply -f - <<EOF
        apiVersion: rbac.authorization.k8s.io/v1
        kind: ClusterRoleBinding
        metadata:
          name: aro-admins
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: ClusterRole
          name: cluster-admin
        subjects:
        - apiGroup: rbac.authorization.k8s.io
          kind: Group
          name: <group-id>
        EOF
        ```

## Validate the external authentication configuration

After setting up external authentication for your cluster, you can validate the configuration for the OpenShift CLI and web console.

### Validate access to the OpenShift web console

1. Retrieve the URL for the OpenShift web console.

    ``` console linenums="0"
    $ az rest --method GET --uri "/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${CUSTOMER_RG_NAME}/providers/Microsoft.RedHatOpenShift/hcpOpenShiftClusters/${CLUSTER_NAME}?api-version=2024-06-10-preview" | jq -r '.properties.console.url'
    ```

1. Open the URL in your web browser and sign in.

### Validate access to the OpenShift CLI

You can use `kubelogin`, a client-go credential plugin, to authenticate to your OpenShift cluster using the external authentication provider (Entra ID).

**Prerequisites**

* Krew (the `kubectl` plugin manager). See the [installation instructions](https://krew.sigs.k8s.io/docs/user-guide/setup/install/).  
* `kubelogin`. See the [installation instructions](https://github.com/int128/kubelogin?tab=readme-ov-file#setup).

To validate access to the CLI:

1. Set the OIDC configuration in the `kubeconfig` that you used to access the cluster.

    ``` console linenums="0"
    $ oc config set-credentials oidc \
      --exec-api-version="client.authentication.k8s.io/v1beta1" \
      --exec-command="kubectl" \
      --exec-arg="oidc-login" \
      --exec-arg="get-token" \
      --exec-arg="--oidc-issuer-url=$ISSUER_URL" \
      --exec-arg="--oidc-client-id=$CLIENT_ID" \
      --exec-arg="--oidc-client-secret=$CLIENT_SECRET"
    ```

1. Create a new context for the OIDC configuration.

    ``` console linenums="0"
    $ oc config set-context oidc --cluster=cluster --namespace=default --user=oidc
    ```

1. Switch to the new context.

    ``` console linenums="0"
    $ oc config use-context oidc
    ```

1. Enter a command and then log in to verify access to the cluster.

    ``` console linenums="0"
    $ oc get nodes --insecure-skip-tls-verify
    ```

1. Verify your credentials.

    ``` console linenums="0"
    $ oc auth whoami --insecure-skip-tls-verify
    ATTRIBUTE    VALUE
    Username     user@email.com
    Groups       [<group-id> system:authenticated]
    ```
