
# Add pull secrets to access private registries

You can add additional pull secrets to access container images from private registries.

1. Create a `DockerConfigJSON` config file that contains your authentication credentials for the private registry.

    !!! example "Example `DockerConfigJSON` file"

        ``` json
        {
            "auths": {
                "registry.example.com": {
                    "auth": "<base64-encoded-credentials>"
                }
            }
        }
        ```

1. Base64-encode the `DockerConfigJSON` file.

1. In your OpenShift cluster, create a `Secret` object to define your pull secret.

    !!! example "Example `Secret` object for a pull secret"

        ``` yaml
        apiVersion: v1
        kind: Secret
        metadata:
            name: <pull-secret-name>
            namespace: kube-system
        type: kubernetes.io/dockerconfigjson
        data:
            .dockerconfigjson: <base64-encoded-docker-config-json>
        ```

1. Apply the secret.

    ``` console linenums="0"
    $ oc apply -f <pull-secret-name>.yaml
    ```

    You can now use the pull secret in a pod or service account.
