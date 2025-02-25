# Install Carbon Black Scanner (Beta)

This topic describes how to install Supply Chain Security Tools - Scan 
(Carbon Black Scanner) from the Tanzu Application Platform package repository.

>**Note:** Carbon Black's image scanning capability is in beta. Carbon Black might only return a partial list of CVEs when scanning Buildpack images.

## <a id="prerecs"></a> Prerequisites

Before installing SCST - Scan (Carbon Black Scanner):

- Install SCST - Scan. See [Install Supply Chain Security Tools - Scan](install-scst-scan.md).
It must be present on the same cluster.
The prerequisites for SCST - Scan are also required.
- Obtain a Carbon Black API Token from Carbon Black Cloud.

## <a id="install-carbonblack"></a> Install

To install SCST - Scan (Carbon Black Scanner):

1. List version information for the package by running:

    ```console
    tanzu package available list carbonblack.scanning.apps.tanzu.vmware.com --namespace tap-install
    ```

    For example:

    ```console
    $ tanzu package available list carbonblack.scanning.apps.tanzu.vmware.com --namespace tap-install
    / Retrieving package versions for carbonblack.scanning.apps.tanzu.vmware.com...
      NAME                                         VERSION       RELEASED-AT
      carbonblack.scanning.apps.tanzu.vmware.com   1.0.0
    ```

2. (Optional) Make changes to the default installation settings by running:

    ```console
    tanzu package available get carbonblack.scanning.apps.tanzu.vmware.com/VERSION --values-schema -n tap-install
    ```

    Where `VERSION` is your package version number. For example, `1.0.0`.

    For example:

    ```console
    $ tanzu package available get carbonblack.scanning.apps.tanzu.vmware.com/1.0.0 --values-schema -n tap-install

    KEY                                           DEFAULT                                                           TYPE    DESCRIPTION
    namespace                                     default                                                           string  Deployment namespace for the Scan Templates
    resources.requests.cpu                        250m                                                              <nil>   Requests describes the minimum amount of cpu resources required.
    resources.requests.memory                     128Mi                                                             <nil>   Requests describes the minimum amount of memory resources required.
    resources.limits.cpu                          1000m                                                             <nil>   Limits describes the maximum amount of cpu resources allowed.
    targetImagePullSecret                                                                                           string  Reference to the secret used for pulling images from private registry.
    carbonBlack.configSecret.name                                                                                   string  Reference to the secret containing the CBC configuration - cbc_org_key, cbc_api_key, cbc_api_id and cbc_saas_url.
    metadataStore.authSecret.importFromNamespace                                                                    string  Namespace from which to import the Insight Metadata Store auth_token
    metadataStore.authSecret.name                                                                                   string  Name of deployed Secret with key auth_token
    metadataStore.caSecret.importFromNamespace    metadata-store                                                    string  Namespace from which to import the Insight Metadata Store CA Cert
    metadataStore.caSecret.name                   app-tls-cert                                                      string  Name of deployed Secret with key ca.crt holding the CA Cert of the Insight Metadata Store
    metadataStore.clusterRole                     metadata-store-read-write                                         string  Name of the deployed ClusterRole for read/write access to the Insight Metadata Store deployed in the same cluster
    metadataStore.url                             https://metadata-store-app.metadata-store.svc.cluster.local:8443  string  Url of the Insight Metadata Store
    ```

3. Create a Carbon Black secret YAML file and insert the Carbon Black API configuration key where:

   - `cbc_api_id` - The API ID obtained from CBC
   - `cbc_api_key` - The API Key obtained from CBC
   - `cbc_org_key` - The Org Key of your CBC organization
   - `cbc_saas_url` - The CBC Backend URL

  > **Note:** All values are obtained from your CBC console.

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: CARBONBLACK-CONFIG-SECRET
      namespace: my-apps
    stringData:
      cbc_api_id: <CBC_API_ID>
      cbc_api_key: <CBC_API_KEY>
      cbc_org_key: <CBC_ORG_KEY>
      cbc_saas_url: <CBC_SAAS_URL>
    ```

1. Apply the Carbon Black secret YAML file by running:

    ```console
    kubectl apply -f YAML-FILE
    ```

    Where `YAML-FILE` is the name of the Carbon Black secret YAML file you created.

2. Define the `--values-file` flag to customize the default configuration. Create a `values.yaml` file by using the following configuration:

    You must define the following text boxes in the `values.yaml` file for the Carbon Black Scanner configuration.
    You can add text boxes as needed to enable or deactivate behaviors.
    You can append the values to this file as shown later in this topic. 

    ```yaml
    ---
    namespace: DEV-NAMESPACE
    targetImagePullSecret: TARGET-REGISTRY-CREDENTIALS-SECRET
    carbonBlack:
      configSecret:
        name: CARBONBLACK-CONFIG-SECRET
    ```

     - `DEV-NAMESPACE` is your developer namespace.

       >**Note:** To use a namespace other than the default namespace, ensure that the namespace exists before you install.
       If the namespace does not exist, the scanner installation fails.

     - `TARGET-REGISTRY-CREDENTIALS-SECRET` is the name of the secret that contains the credentials to pull an image from a private registry for scanning.

     - `CARBONBLACK-CONFIG-SECRET` is the name of the secret you created that contains the Carbon Black configuration to connect to CBC.
       This text box is required.

    The Carbon Black Scanner integration can work with or without the SCST - Store integration.
    The `values.yaml` file is slightly different for each configuration.

### <a id="using-integration"></a> Using SCST - Store Integration 

To persist the results found by the Carbon Black Scanner, you can enable the SCST - Store integration by appending the text boxes to the `values.yaml` file. 

The Grype and Carbon Black Scanner Integrations both enable the Metadata Store.
To prevent conflicts, the configuration values are slightly different based on whether the Grype Scanner Integration is installed or not.
If Tanzu Application Platform was installed using the Full Profile, the Grype Scanner Integration was installed, unless it was explicitly excluded.

  * If the Grype Scanner Integration is installed in the same `dev-namespace` Carbon Black Scanner is installed:

       ```yaml
       #! ...
       metadataStore:
         #! The url where the Store deployment is accesible.
         #! Default value is: "https://metadata-store-app.metadata-store.svc.cluster.local:8443"
         url: "<STORE-URL>" 
         caSecret:
           #! The name of the secret that contains the ca.crt to connect to the Store Deployment.
           #! Default value is: "app-tls-cert"
           name: "<CA-SECRET-NAME>"
           importFromNamespace: "" #! since both Carbon Black and Grype both enable store, one must leave importFromNamespace blank
         #! authSecret is for multicluster configurations.
         authSecret:
           #! The name of the secret that contains the auth token to authenticate to the Store Deployment.
           name: "<AUTH-SECRET-NAME>"
           importFromNamespace: "" #! since both Carbon Black and Grype both enable store, one must leave importFromNamespace blank
       ```

  * If the Grype Scanner Integration is not installed in the same `dev-namespace` Carbon Black Scanner is installed:

       ```yaml
       #! ...
       metadataStore:
         #! The url where the Store deployment is accesible.
         #! Default value is: "https://metadata-store-app.metadata-store.svc.cluster.local:8443"
         url: "<STORE-URL>" 
         caSecret:
           #! The name of the secret that contains the ca.crt to connect to the Store Deployment.
           #! Default value is: "app-tls-cert"
           name: "<CA-SECRET-NAME>"
           #! The namespace where the secrets for the Store Deployment live. 
           #! Default value is: "metadata-store"
           importFromNamespace: "<STORE-SECRETS-NAMESPACE>"
         #! authSecret is for multicluster configurations.
         authSecret:
           #! The name of the secret that contains the auth token to authenticate to the Store Deployment.
           name: "<AUTH-SECRET-NAME>"
           #! The namespace where the secrets for the Store Deployment live.
           importFromNamespace: "<STORE-SECRETS-NAMESPACE>"
       ```

### <a id="without-scst"></a> Without SCST - Store Integration

If you don't want to enable the SCST - Store integration, explicitly deactivate the integration by appending the next text box to the `values.yaml` file, because it's enabled by default: 

  ```yaml
  # ... 
  metadataStore:
    url: "" # Disable Supply Chain Security Tools - Store integration
  ```

1. Install the package by running:

    ```console
    tanzu package install carbonblack-scanner \
      --package-name carbonblack.scanning.apps.tanzu.vmware.com \
      --version VERSION \
      --namespace tap-install \
      --values-file values.yaml
    ```

    Where `VERSION` is your package version number. For example, `1.0.0`.

    For example:

    ```console
    $ tanzu package install carbonblack-scanner \
      --package-name carbonblack.scanning.apps.tanzu.vmware.com \
      --version 1.1.0 \
      --namespace tap-install \
      --values-file values.yaml
    / Installing package 'carbonblack.scanning.apps.tanzu.vmware.com'
    | Getting namespace 'tap-install'
    | Getting package metadata for 'carbonblack.scanning.apps.tanzu.vmware.com'
    | Creating service account 'carbonblack-scanner-tap-install-sa'
    | Creating cluster admin role 'carbonblack-scanner-tap-install-cluster-role'
    | Creating cluster role binding 'carbonblack-scanner-tap-install-cluster-rolebinding'
    / Creating package resource
    - Package install status: Reconciling

     Added installed package 'carbonblack-scanner' in namespace 'tap-install'
    ```

## <a id="verify"></a> Verify integration with Carbon Black

To verify the integration with Carbon Black, apply the following `ImageScan` and its `ScanPolicy` in the developer namespace and review the result.

1. Create a ScanPolicy YAML with a Rego file for scanner output in the CycloneDX format. Here is a sample scan policy resource:

    ```yaml
    apiVersion: scanning.apps.tanzu.vmware.com/v1beta1
    kind: ScanPolicy
    metadata:
      name: carbonblack-scan-policy
      labels:
        'app.kubernetes.io/part-of': 'component-a'
    spec:
      regoFile: |
        package main

        # Accepted Values: "Critical", "High", "Medium", "Low", "Negligible", "UnknownSeverity"
        notAllowedSeverities := ["Critical","High"]
        ignoreCves := []

        contains(array, elem) = true {
          array[_] = elem
        } else = false { true }

        isSafe(match) {
          severities := { e | e := match.ratings.rating.severity } | { e | e := match.ratings.rating[_].severity }
          some i
          fails := contains(notAllowedSeverities, severities[i])
          not fails
        }

        isSafe(match) {
          ignore := contains(ignoreCves, match.id)
          ignore
        }

        deny[msg] {
          comps := { e | e := input.bom.components.component } | { e | e := input.bom.components.component[_] }
          some i
          comp := comps[i]
          vulns := { e | e := comp.vulnerabilities.vulnerability } | { e | e := comp.vulnerabilities.vulnerability[_] }
          some j
          vuln := vulns[j]
          ratings := { e | e := vuln.ratings.rating.severity } | { e | e := vuln.ratings.rating[_].severity }
          not isSafe(vuln)
          msg = sprintf("CVE %s %s %s", [comp.name, vuln.id, ratings])
        }
    ```

2. Apply the earlier created YAML:

    ```console
    kubectl apply -n $DEV_NAMESPACE -f <SCAN-POLICY-YAML>
    ```

3. Create the following ImageScan YAML:

    ```yaml
    apiVersion: scanning.apps.tanzu.vmware.com/v1beta1
    kind: ImageScan
    metadata:
      name: sample-carbonblack-public-image-scan
    spec:
      registry:
        image: "nginx:1.16"
      scanTemplate: carbonblack-public-image-scan-template
      scanPolicy: carbonblack-scan-policy
    ```

4. Apply the earlier created YAML:

    ```console
    kubectl apply -n $DEV_NAMESPACE -f <IMAGE-SCAN-YAML>
    ``` 

5. To verify the integration, run:

    ```bash
    kubectl get imagescan sample-carbonblack-public-image-scan -n $DEV_NAMESPACE
    ```

    For example:

    ```console
    kubectl get imagescan sample-carbonblack-public-image-scan -n $DEV_NAMESPACE
    NAME                                   PHASE       SCANNEDIMAGE   AGE   CRITICAL   HIGH   MEDIUM   LOW   UNKNOWN   CVETOTAL
    sample-carbonblack-public-image-scan   Completed   nginx:1.16     26h   0          114    58       314   0         486
    ```

6. Cleanup:

    ```bash
    kubectl delete imagescan sample-carbonblack-public-image-scan -n $DEV_NAMESPACE
    ```

## <a id="sc-config"></a> Configure Supply Chains

In order to scan your images with Carbon Black instead of the default Grype scanner in the [Out of the Box Supply Chain with Testing and Scanning](../scc/ootb-supply-chain-testing-scanning.md), you must update your Tanzu Application Platform installation.
 
Add the `ootb_supply_chain_testing_scanning.scanning` section later to your `tap-values.yaml` and perform a [Tanzu Application Platform update](../upgrading.md#upgrading-tanzu-application-platform).

```yaml
ootb_supply_chain_testing_scanning:
  scanning:
    image:
      template: carbonblack-private-image-scan-template
      policy: carbonblack-scan-policy
```

>**Note:** The Carbon Black Scanner integration is only available for an image scan, not a source scan.

## <a id="opt-out-of-carbonblack"></a> Opt-out of using Carbon Black

You can opt out of using Carbon Black for either a specific supply chain or for all of Tanzu Application Platform.

### <a id="opt-out-of-carbonblack-supply-chain"></a> Opt-out of Carbon Black for a Supply Chain

To opt-out of Carbon Black for a specific Supply Chain, reconfigure the supply chain to use another scanner:

-  Edit the `ootb_supply_chain_testing_scanning.scanning.image.template` value to use a scan template that does not use Carbon Black, such as Grype.

      ```yaml
      ootb_supply_chain_testing_scanning:
        scanning:
          image:
            template: "ALTERNATIVE-SCAN-TEMPLATE"
            policy: carbonblack-scan-policy
      ```

### <a id="opt-out-of-carbonblack-entirely"></a> Opt-out of Carbon Black Entirely

To opt-out of Carbon Black for all of Tanzu Application Platform:

1. To uninstall Carbon Black, run:

  ```console
  tanzu package installed delete carbonblack-scanner \
    --namespace tap-install
  ```

2. Follow the [Opt-out of Carbon Black for a specific Supply Chain](#-opt-out-of-carbonblack-for-a-supply-chain) for all Supply Chains in the environment to not use Carbon Black and use another scanner such as Grype.
