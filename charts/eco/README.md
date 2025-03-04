# eco

![Version: 0.0.4](https://img.shields.io/badge/Version-0.0.4-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: eco.0.0.1](https://img.shields.io/badge/AppVersion-eco.0.0.1-informational?style=flat-square)

A Helm chart for Pharma Ledger eco applications

## Requirements

- [helm 3](https://helm.sh/docs/intro/install/)
- These mandatory configuration values:
  - Domain - The Domain - e.g. `eco`
  - Sub Domain - The Sub Domain - e.g. `eco.my-company`
  - Vault Domain - The Vault Domain - e.g. `vault.my-company`
  - ethadapterUrl - The Full URL of the Ethadapter including protocol and port -  e.g. "https://ethadapter.my-company.com:3000"
  - bdnsHosts - The Centrally managed and provided BDNS Hosts Config -

## Usage

- [Here](./README.md#values) is a full list of all configuration values.
- The [values.yaml file](./values.yaml) shows the raw view of all configuration values.

## Helm Lifecycle and Kubernetes Resources Lifetime

This helm chart uses Helm [hooks](https://helm.sh/docs/topics/charts_hooks/) in order to install, upgrade and manage the application and its resources.

```mermaid
sequenceDiagram
  participant PIN as pre-install
  participant PUP as pre-upgrade
  participant I as install
  participant U as uninstall
  participant PUN as post-uninstall
  Note over PIN,PUN: PersistentVolumeClaim
  Note over PIN,PUN: ConfigMap SeedsBackup
  Note over PIN:Init Job
  Note over PIN:ConfigMaps Init
  Note over PIN:ServiceAccount Init
  Note over PIN:Role Init
  Note over PIN:RoleBinding Init
  note right of PIN: Note: The Init Job stores <br/>Seeds in Configmap SeedsBackup and <br/> is either executed by a) pre-install hook or<br/>b)pre-upgrade hook
  Note over PUP,U:Deployment
  Note over PUP,U:ConfigMap build-info
  Note over PUP,U:Configmaps for application
  Note over PUP,U:Service
  Note over PUP,U:Ingress
  Note over PUP,U:ServiceAccount
  Note over PUP:Init Job<br/>and more<br/>(see pre-install)
  Note over PUN:Cleanup Job
  Note over PUN:ServiceAccount Cleanup
  Note over PUN:Role Cleanup
  Note over PUN:RoleBinding Cleanup
  note right of PUN: Note: The Cleanup job<br/>1. deletes PersistentVolumeClaim (optional)<br/>2. creates final backup of ConfigMap SeedsBackup<br/>3. deletes ConfigMap SeedsBackup
```

## Init Job

The Init Job is required to run the build process and to store the SeedsBackup in a ConfigMap.

- The Init Job will be executed on helm [hooks](https://helm.sh/docs/topics/charts_hooks/) `pre-install` and `pre-upgrade`.
- It is only necessary to run/deploy the Init Job on installation and on subsequent software changes (=helm upgrade in combination with use of a different image than before).
- Therefore the Init Job will only be deployed on installation and on helm upgrades if the image has changed.

### Init Job Details

The pod consists an init containers and a main container.

1. The Init Container uses the container image of the eco application and
   1. Starts the apihub server (`npm run server`), waits for a short period of time and then starts the build process (`npm run build-all`).
   2. After build process, it writes the SeedsBackup file on a shared temporary volume between init and main container.

    ```mermaid
    flowchart LR
    A(Init Container<br/>application) --> B[start apihub server]
    B --> C[sleep short time]
    C --> D[build process]
    D --> E[write SeedsBackup file to shared data with main container]
    E --> F[Exit Init Container<br/>application]
    ```

2. The Main Container has `kubectl` installed and checks if SeedsBackup file was handed over by Init Container.

    ```mermaid
    flowchart LR
    G(Main Container) --> H[Create ConfigMap SeedsBackup for current Image]
    H --> I[Update ConfigMap SeedsBackup]
    I --> J[Exit Pod]
    ```

After completion of the *Init Job* the application container will be deployed/restarted with the current *ConfigMap SeedsBackup*.

## Cleanup Job

On deletion/uninstall of the helm chart a Kubernetes `cleanup` will be deployed in order to delete unmanaged helm resources created by helm hooks at `pre-install`.
These resources are:

1. Init Job - The Init Job was created on pre-install/pre-upgrade and will remain after its execution.
2. PersistentVolumeClaim - In case the PersistentVolumeClaim shall not be deleted on deletion of the helm release, set `persistence.deletePvcOnUninstall` to `false`.
3. ConfigMap SeedsBackup - Prior to deletion of the ConfigMap, a backup ConfigMap will be created with naming schema `{HELM_RELEASE_NAME}-seedsbackup-{IMAGE_TAG}-final-backup-{EPOCH_IN_SECONDS}`, e.g. `eco-seedsbackup-poc.1.6-final-backup-1646063552`

## Installation

### Quick install with internal service of type ClusterIP

By default, this helm chart installs the eco use case at an internal ClusterIP Service.
This is to prevent exposing the service to the internet by accident!

It is recommended to put non-sensitive configuration values in an configuration file and pass sensitive/secret values via commandline.

1. Create configuration file, e.g. *my-config.yaml*

    ```yaml
    config:
      domain: "domain_value"
      subDomain: "subDomain_value"
      vaultDomain: "vaultDomain_value"
      ethadapterUrl: "https://ethadapter.my-company.com:3000"
      bdnsHosts: |-
        # ... content of the BDNS Hosts file ...

    ```

2. Install via helm to namespace `default`

    ```bash
    helm upgrade my-release-name pharmaledger-imi/eco --version=0.0.4 \
        --install \
        --values my-config.yaml \
    ```

### Expose Service via Load Balancer

In order to expose the service **directly** by an **own dedicated** Load Balancer, just **add** `service.type` with value `LoadBalancer` to your config file (in order to override the default value which is `ClusterIP`).

**Please note:** At AWS using `service.type` = `LoadBalancer` is not recommended any more, as it creates a Classic Load Balancer. Use [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/) with an ingress instead. A full sample is provided later in the docs. Using an Application Load Balancer (managed by AWS LB Controller) increases security (e.g. by using a Web Application Firewall for your http based traffic) and provides more features like hostname, pathname routing or built-in authentication mechanism via OIDC or AWS Cognito.

Configuration file *my-config.yaml*

```yaml
service:
  type: LoadBalancer

config:
  # ... config section keys and values ...
```

There are more configuration options available like customizing the port and configuring the Load Balancer via annotations (e.g. for configuring SSL Listener).

**Also note:** Annotations are very specific to your environment/cloud provider, see [Kubernetes Service Reference](https://kubernetes.io/docs/concepts/services-networking/service/#ssl-support-on-aws) for more information. For Azure, take a look [here](https://kubernetes-sigs.github.io/cloud-provider-azure/topics/loadbalancer/#loadbalancer-annotations).

Sample for AWS (SSL and listening on port 1234 instead 80 which is the default):

```yaml
service:
  type: LoadBalancer
  port: 80
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: http
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "80"
    # https://docs.aws.amazon.com/de_de/elasticloadbalancing/latest/classic/elb-security-policy-table.html
    service.beta.kubernetes.io/aws-load-balancer-ssl-negotiation-policy: "ELBSecurityPolicy-TLS-1-2-2017-01"

# further config
```

### AWS Load Balancer Controller: Expose Service via Ingress

Note: You need the [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/) installed and configured properly.

1. Enable ingress
2. Add *host*, *path* *`/*`* and *pathType* `ImplementationSpecific`
3. Add annotations for AWS LB Controller
4. A SSL certificate at AWS Certificate Manager (either for the hostname, here `eco.mydomain.com` or wildcard `*.mydomain.com`)

Configuration file *my-config.yaml*

```yaml
ingress:
  enabled: true
  # Let AWS LB Controller handle the ingress (default className is alb)
  # Note: Use className instead of annotation 'kubernetes.io/ingress.class' which is deprecated since 1.18
  # For Kubernetes >= 1.18 it is required to have an existing IngressClass object.
  # See: https://kubernetes.io/docs/concepts/services-networking/ingress/#deprecated-annotation
  className: alb
  hosts:
    - host: eco.mydomain.com
      # Path must be /* for ALB to match all paths
      paths:
        - path: /*
          pathType: ImplementationSpecific
  # For full list of annotations for AWS LB Controller, see https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/guide/ingress/annotations/
  annotations:
    # The ARN of the existing SSL Certificate at AWS Certificate Manager
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:REGION:ACCOUNT_ID:certificate/CERTIFICATE_ID
    # The name of the ALB group, can be used to configure a single ALB by multiple ingress objects
    alb.ingress.kubernetes.io/group.name: default
    # Specifies the HTTP path when performing health check on targets.
    alb.ingress.kubernetes.io/healthcheck-path: /
    # Specifies the port used when performing health check on targets.
    alb.ingress.kubernetes.io/healthcheck-port: traffic-port
    # Specifies the HTTP status code that should be expected when doing health checks against the specified health check path.
    alb.ingress.kubernetes.io/success-codes: "200"
    # Listen on HTTPS protocol at port 443 at the ALB
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    # Use internet facing
    alb.ingress.kubernetes.io/scheme: internet-facing
    # Use most current (as of Dec 2021) encryption ciphers
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-2-Ext-2018-06
    # Use target type IP which is the case if the service type is ClusterIP
    alb.ingress.kubernetes.io/target-type: ip

config:
  # ... config section keys and values ...
```

## Additional helm options

Run `helm upgrade --helm` for full list of options.

1. Install to other namespace

    You can install into other namespace than `default` by setting the `--namespace` parameter, e.g.

    ```bash
    helm upgrade my-release-name pharmaledger-imi/eco --version=0.0.4 \
        --install \
        --namespace=my-namespace \
        --values my-config.yaml \
    ```

2. Wait until installation has finished successfully and the deployment is up and running.

    Provide the `--wait` argument and time to wait (default is 5 minutes) via `--timeout`

    ```bash
    helm upgrade my-release-name pharmaledger-imi/eco --version=0.0.4 \
        --install \
        --wait --timeout=600s \
        --values my-config.yaml \
    ```

## Potential issues

1. `Error: admission webhook "vingress.elbv2.k8s.aws" denied the request: invalid ingress class: IngressClass.networking.k8s.io "alb" not found`

    **Description:** This error only applies to Kubernetes >= 1.18 and indicates that no matching *IngressClass* object was found.

    **Solution:** Either declare an appropriate IngressClass or omit *className* and add annotation `kubernetes.io/ingress.class`

    Further information:

     - [Kubernetes IngressClass](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-class)
     - [AWS Load Balancer controller documentation](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/guide/ingress/ingress_class/)

## Helm Unittesting

[helm-unittest](https://github.com/quintush/helm-unittest) is being used for testing the output of the helm chart.
Tests can be found in [tests](./tests)

## Maintainers

| Name | Email | Url |
| ---- | ------ | --- |
| tgip-work |  | <https://github.com/tgip-work> |

## Values

*Note:* Please scroll horizontally to show more columns (e.g. description)!

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| affinity | object | `{}` | Affinity for scheduling a pod. See [https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) |
| apiHubWorkingFolder | string | `"eco-workspace"` |  |
| config.apihub | string | `"{\n  \"storage\": \"../apihub-root\",\n  \"port\": 8080,\n  \"preventRateLimit\": true,\n  \"activeComponents\": [\n    \"virtualMQ\",\n    \"messaging\",\n    \"notifications\",\n    \"filesManager\",\n    \"bdns\",\n    \"bricksLedger\",\n    \"bricksFabric\",\n    \"bricking\",\n    \"anchoring\",\n    \"debugLogger\",\n    \"mq\",\n    \"staticServer\"\n  ],\n  \"componentsConfig\": {\n    \"staticServer\": {\n                   \"excludedFiles\": [\n                       \".*.secret\"\n                   ]\n               },\n    \"bricking\": {},\n    \"anchoring\": {}\n  },\n  \"responseHeaders\": {\n             \"X-Frame-Options\": \"SAMEORIGIN\",\n             \"X-XSS-Protection\": \"1; mode=block\"\n         },\n  \"enableRequestLogger\": true,\n  \"enableJWTAuthorisation\": false,\n  \"enableLocalhostAuthorization\": false,\n  \"serverAuthentication\": false,\n  \"skipJWTAuthorisation\": [\n    \"/assets\",\n    \"/directory-summary\",\n    \"/resources\",\n    \"/bdns\",\n    \"/anchor/default\",\n    \"/anchor/vault\",\n    \"/bricking\",\n    \"/bricksFabric\",\n    \"/bricksledger\",\n    \"/create-channel\",\n    \"/forward-zeromq\",\n    \"/send-message\",\n    \"/receive-message\",\n    \"/files\",\n    \"/notifications\",\n    \"/mq\"\n  ]\n}"` | Configuration file apihub.json. Settings: [https://docs.google.com/document/d/1mg35bb1UBUmTpL1Kt4GuZ7P0K_FMqt2Mb8B3iaDf52I/edit#heading=h.z84gh8sclah3](https://docs.google.com/document/d/1mg35bb1UBUmTpL1Kt4GuZ7P0K_FMqt2Mb8B3iaDf52I/edit#heading=h.z84gh8sclah3) <br/> For epi <= v1.1.2: Replace "module": "./../../gtin-resolver" with "module": "./../../epi-utils" <br/> For SSO (not enabled by default!): <br/> 1. "enableOAuth": true <br/> 2. "serverAuthentication": true <br/> 3. For SSO via OAuth with Azure AD, replace <TODO_*> with appropriate values.    For other identity providers (IdP) (e.g. Google, Ping, 0Auth), refer to documentation.    "redirectPath" must match the redirect URL configured at IdP <br/> 4. Add these values to "skipOAuth": "/leaflet-wallet/", "/directory-summary/", "/iframe/" |
| config.bdnsHosts | string | `"{\n\"default\": {\n  \"replicas\": [],\n  \"brickStorages\": [\n    \"$ORIGIN\"\n  ],\n  \"anchoringServices\": [\n    \"$ORIGIN\"\n  ]\n},\n\"vault.my-company\": {\n  \"replicas\": [],\n  \"brickStorages\": [\n    \"$ORIGIN\"\n  ],\n  \"anchoringServices\": [\n    \"$ORIGIN\"\n  ]\n},\n\"eco.my-company\":{\n    \"replicas\":[],\n    \"brickStorages\":[\n       \"$ORIGIN\"\n    ],\n    \"anchoringServices\":[\n       \"$ORIGIN\"\n    ]\n },\n\"eco\":{\n        \"replicas\":[],\n        \"brickStorages\":[\n           \"$ORIGIN\"\n        ],\n        \"anchoringServices\":[\n           \"$ORIGIN\"\n        ]\n},\n\"iot\":{\n  \"replicas\":[\n\n  ],\n  \"brickStorages\":[\n     \"http://localhost:8080\"\n  ],\n  \"anchoringServices\":[\n     \"http://localhost:8080\"\n  ],\n  \"mqEndpoints\":[\n     \"http://localhost:8080\"\n  ]\n}\n}"` | Centrally managed and provided BDNS Hosts Config |
| config.demiurgeMode | string | `"dev-secure"` |  |
| config.domain | string | `"eco"` | The Domain, e.g. "epipoc" |
| config.dsuFabricMode | string | `"dev-secure"` |  |
| config.ethadapterUrl | string | `"http://ethadapter.ethadapter:3000"` | The Full URL of the Ethadapter including protocol and port, e.g. "https://ethadapter.my-company.com:3000" |
| config.sleepTime | string | `"10s"` |  |
| config.subDomain | string | `"eco.my-company"` | The Subdomain, should be domain.company, e.g. epipoc.my-company |
| config.vaultDomain | string | `"vault.my-company"` | The Vault domain, should be vault.company, e.g. vault.my-company |
| deploymentStrategy | object | `{"type":"Recreate"}` | The strategy of the deployment. Defaults to type: Recreate as a PVC is bound to it. See `kubectl explain deployment.spec.strategy` for more and [https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy) |
| fullnameOverride | string | `""` | fullnameOverride completely replaces the generated name. From [https://stackoverflow.com/questions/63838705/what-is-the-difference-between-fullnameoverride-and-nameoverride-in-helm](https://stackoverflow.com/questions/63838705/what-is-the-difference-between-fullnameoverride-and-nameoverride-in-helm) |
| image.pullPolicy | string | `"IfNotPresent"` | Image Pull Policy |
| image.repository | string | `"username/eco-docker-image"` | The repository of the container image |
| image.sha | string | `""` | sha256 digest of the image. Do not add the prefix "@sha256:" |
| image.tag | string | `"0.0.1"` | Overrides the image tag whose default is the chart appVersion. |
| imagePullSecrets | list | `[]` | Secret(s) for pulling an container image from a private registry. See [https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/) |
| ingress.annotations | object | `{}` | Ingress annotations. <br/> For AWS LB Controller, see [https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/guide/ingress/annotations/](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/guide/ingress/annotations/) <br/> For Azure Application Gateway Ingress Controller, see [https://azure.github.io/application-gateway-kubernetes-ingress/annotations/](https://azure.github.io/application-gateway-kubernetes-ingress/annotations/) <br/> For NGINX Ingress Controller, see [https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/) <br/> For Traefik Ingress Controller, see [https://doc.traefik.io/traefik/routing/providers/kubernetes-ingress/#annotations](https://doc.traefik.io/traefik/routing/providers/kubernetes-ingress/#annotations) |
| ingress.className | string | `""` | The className specifies the IngressClass object which is responsible for that class. See [https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-class](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-class) <br/> For Kubernetes >= 1.18 it is required to have an existing IngressClass object. If IngressClass object does not exists, omit className and add the deprecated annotation 'kubernetes.io/ingress.class' instead. <br/> For Kubernetes < 1.18 either use className or annotation 'kubernetes.io/ingress.class'. |
| ingress.enabled | bool | `false` | Whether to create ingress or not. <br/> Note: For ingress an Ingress Controller (e.g. AWS LB Controller, NGINX Ingress Controller, Traefik, ...) is required and service.type should be ClusterIP or NodePort depending on your configuration |
| ingress.hosts | list | `[{"host":"eco.some-pharma-company.com","paths":[{"path":"/","pathType":"ImplementationSpecific"}]}]` | A list of hostnames and path(s) to listen at the Ingress Controller |
| ingress.hosts[0].host | string | `"eco.some-pharma-company.com"` | The FQDN/hostname |
| ingress.hosts[0].paths[0].path | string | `"/"` | The Ingress Path. See [https://kubernetes.io/docs/concepts/services-networking/ingress/#examples](https://kubernetes.io/docs/concepts/services-networking/ingress/#examples) <br/> Note: For Ingress Controllers like AWS LB Controller see their specific documentation. |
| ingress.hosts[0].paths[0].pathType | string | `"ImplementationSpecific"` | The type of path. This value is required since Kubernetes 1.18. <br/> For Ingress Controllers like AWS LB Controller or Traefik it is usually required to set its value to ImplementationSpecific See [https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types](https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types) and [https://kubernetes.io/blog/2020/04/02/improvements-to-the-ingress-api-in-kubernetes-1.18/](https://kubernetes.io/blog/2020/04/02/improvements-to-the-ingress-api-in-kubernetes-1.18/) |
| ingress.tls | list | `[]` |  |
| kubectl.image.pullPolicy | string | `"IfNotPresent"` | Image Pull Policy |
| kubectl.image.repository | string | `"bitnami/kubectl"` | The repository of the container image containing kubectl |
| kubectl.image.sha | string | `"f9814e1d2f1be7f7f09addd1d877090fe457d5b66ca2dcf9a311cf1e67168590"` | sha256 digest of the image. Do not add the prefix "@sha256:" <br/> Defaults to image digest for "bitnami/kubectl:1.21.8", see [https://hub.docker.com/layers/kubectl/bitnami/kubectl/1.21.8/images/sha256-f9814e1d2f1be7f7f09addd1d877090fe457d5b66ca2dcf9a311cf1e67168590?context=explore](https://hub.docker.com/layers/kubectl/bitnami/kubectl/1.21.8/images/sha256-f9814e1d2f1be7f7f09addd1d877090fe457d5b66ca2dcf9a311cf1e67168590?context=explore) <!-- # pragma: allowlist secret --> |
| kubectl.image.tag | string | `"1.21.8"` | The Tag of the image containing kubectl. Minor Version should match to your Kubernetes Cluster Version. |
| livenessProbe | object | `{"failureThreshold":3,"httpGet":{"path":"/","port":"http"},"initialDelaySeconds":10,"periodSeconds":10,"successThreshold":1,"timeoutSeconds":1}` | Liveness probe. See [https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) |
| nameOverride | string | `""` | nameOverride replaces the name of the chart in the Chart.yaml file, when this is used to construct Kubernetes object names. From [https://stackoverflow.com/questions/63838705/what-is-the-difference-between-fullnameoverride-and-nameoverride-in-helm](https://stackoverflow.com/questions/63838705/what-is-the-difference-between-fullnameoverride-and-nameoverride-in-helm) |
| nodeSelector | object | `{}` | Node Selectors in order to assign pods to certain nodes. See [https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) |
| persistence.accessModes | list | `["ReadWriteOnce"]` | AccessModes for the PVC. See [https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes) |
| persistence.dataSource | object | `{}` | DataSource option for cloning an existing volume or creating from a snapshot. See [values.yaml](values.yaml) for more details. |
| persistence.deletePvcOnUninstall | bool | `true` | Boolean flag whether to delete the persistent volume on uninstall or not. |
| persistence.finalizers | list | `["kubernetes.io/pvc-protection"]` | Finalizers for the PVC. See [https://kubernetes.io/docs/concepts/storage/persistent-volumes/#storage-object-in-use-protection](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#storage-object-in-use-protection) |
| persistence.selectorLabels | object | `{}` | Selector Labels for the logs PVC. See [https://kubernetes.io/docs/concepts/storage/persistent-volumes/#selector](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#selector) |
| persistence.size | string | `"6Gi"` | Size of the volume. |
| persistence.storageClassName | string | `""` | Name of the storage class for the PVC. If empty or not set then storage class will not be set - which means that the default storage class will be used. |
| podAnnotations | object | `{}` | Annotations added to the pod |
| podSecurityContext | object | `{}` | Security Context for the pod. IMPORTANT: Take a look at README.md for configuration for non-root user! See [https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-pod](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-pod) <br/> For running as non-root with uid 1000, remove {} from next line and uncomment fsGroup and runAsUser! |
| readinessProbe | object | `{"exec":{"command":["cat","/eco-workspace/apihub-root/ready"]},"failureThreshold":60,"initialDelaySeconds":30,"periodSeconds":5,"successThreshold":1}` | Readiness probe. See [https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) |
| replicaCount | int | `1` | The number of replicas if autoscaling is false |
| resources | object | `{}` | Resource constraints for the container |
| securityContext | object | `{}` | Security Context for the application container IMPORTANT: Take a look at README.md file for configuration for non-root user! See [https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-container) <br/> For running as non-root with uid 1000, remove {} from next line and uncomment next lines! |
| service.annotations | object | `{}` | Annotations for the service. See AWS, see [https://kubernetes.io/docs/concepts/services-networking/service/#ssl-support-on-aws](https://kubernetes.io/docs/concepts/services-networking/service/#ssl-support-on-aws) For Azure, see [https://kubernetes-sigs.github.io/cloud-provider-azure/topics/loadbalancer/#loadbalancer-annotations](https://kubernetes-sigs.github.io/cloud-provider-azure/topics/loadbalancer/#loadbalancer-annotations) |
| service.port | int | `80` | Port where the service will be exposed |
| service.type | string | `"NodePort"` | Either ClusterIP, NodePort or LoadBalancer. See [https://kubernetes.io/docs/concepts/services-networking/service/](https://kubernetes.io/docs/concepts/services-networking/service/) |
| serviceAccount.annotations | object | `{}` | Annotations to add to the service account |
| serviceAccount.automountServiceAccountToken | bool | `false` | Whether automounting API credentials for a service account is enabled or not. See [https://docs.bridgecrew.io/docs/bc_k8s_35](https://docs.bridgecrew.io/docs/bc_k8s_35) |
| serviceAccount.create | bool | `false` | Specifies whether a service account should be created |
| serviceAccount.name | string | `""` | The name of the service account to use. If not set and create is true, a name is generated using the fullname template |
| tolerations | list | `[]` | Tolerations for scheduling a pod. See [https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) |

----------------------------------------------
Autogenerated from chart metadata using [helm-docs v1.10.0](https://github.com/norwoodj/helm-docs/releases/v1.10.0)
