# unichart

> Generic reusable Helm chart for deploying applications on Kubernetes.

unichart is a single, opinionated Helm chart that covers the full lifecycle of a typical Kubernetes workload — from the `Deployment` and `Service` all the way to autoscaling, secrets management, observability, and backup. Instead of writing a bespoke chart per application, teams can use unichart as a shared base and only override the values they need.

## Installation

```bash
helm install my-app oci://ghcr.io/sondn98/unichart/generic \
  --set deployment.image.repository=docker.io/nginx \
  --set deployment.image.tag=1.25
```

Or with a values file:

```bash
helm install my-app oci://ghcr.io/sondn98/unichart/generic -f my-values.yaml
```

See the [`examples/`](./examples) directory for ready-to-use values files.

---

## Configuration

All configuration is done through `values.yaml`. The tables below document every available key, grouped by feature area.

Column meanings:
- **Key** — dot-notation path in `values.yaml`
- **Type** — YAML type (`string`, `bool`, `int`, `object`, `list`)
- **Default** — value shipped in `values.yaml`
- **Example** — a realistic value when the default is empty/null
- **Required** — whether the chart will fail without this key
- **Description** — what the key does

---

### Global Parameters

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `applicationName` | string | `""` | `my-app` | No | Application name. Defaults to `{{ .Chart.Name }}` when empty. Used as the base name for all generated resources. |
| `namespaceOverride` | string | `""` | `production` | No | Override the namespace for all resources. Defaults to the release namespace. |
| `componentOverride` | string | `""` | `frontend` | No | Override the `app.kubernetes.io/component` label on all resources. |
| `partOfOverride` | string | `""` | `my-platform` | No | Override the `app.kubernetes.io/part-of` label on all resources. |
| `additionalLabels` | object | `{}` | `team: backend` | No | Extra labels applied to every resource created by the chart. Supports Helm templating. |
| `extraObjects` | list/object | `[]` | _(see below)_ | No | Arbitrary Kubernetes manifests to render alongside the chart's own resources. Supports Helm templating. Each entry can be a YAML object or a multiline string. See [Extra Objects](#extra-objects) for details. |

---

### Deployment Parameters

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `deployment.enabled` | bool | `true` | — | No | Create a `Deployment` resource. |
| `deployment.image.repository` | string | `""` | `docker.io/nginx` | **Yes** | Container image repository. The chart will fail if this is empty. |
| `deployment.image.tag` | string | `""` | `1.25` | **Yes** | Container image tag. The chart will fail if this is empty. |
| `deployment.image.digest` | string | `""` | `sha256:abc123` | No | Image digest. When non-empty, takes precedence over `tag`. |
| `deployment.image.pullPolicy` | string | `IfNotPresent` | — | No | Image pull policy. One of `Always`, `IfNotPresent`, `Never`. |
| `deployment.replicas` | int | `nil` | `3` | No | Number of pod replicas. When unset, the field is omitted (useful with HPA). |
| `deployment.revisionHistoryLimit` | int | `2` | — | No | Number of old ReplicaSets to retain for rollback. |
| `deployment.strategy.type` | string | `RollingUpdate` | — | No | Deployment strategy. One of `RollingUpdate` or `Recreate`. |
| `deployment.reloadOnChange` | bool | `true` | — | No | Adds the `reloader.stakater.com/auto: "true"` annotation so that [Reloader](https://github.com/stakater/Reloader) restarts the pod when a mounted ConfigMap or Secret changes. |
| `deployment.annotations` | object | `nil` | `deploy.example.com/env: prod` | No | Annotations added to the `Deployment` object. |
| `deployment.additionalLabels` | object | `nil` | `tier: web` | No | Extra labels added to the `Deployment` object. |
| `deployment.podLabels` | object | `nil` | `sidecar.istio.io/inject: "true"` | No | Extra labels added to the pod template. Also used in the `Service` selector. |
| `deployment.additionalPodAnnotations` | object | `nil` | `prometheus.io/scrape: "true"` | No | Extra annotations added to the pod template. |
| `deployment.nodeSelector` | object | `nil` | `kubernetes.io/os: linux` | No | Node selector for pod scheduling. |
| `deployment.tolerations` | list | `nil` | _(see example)_ | No | Taint tolerations for the pods. |
| `deployment.affinity` | object | `nil` | _(see example)_ | No | Affinity and anti-affinity rules for the pods. |
| `deployment.topologySpreadConstraints` | list | `nil` | _(see example)_ | No | Topology spread constraints for distributing pods across nodes/zones. |
| `deployment.priorityClassName` | string | `""` | `high-priority` | No | Priority class for the pods. |
| `deployment.runtimeClassName` | string | `""` | `gvisor` | No | Runtime class for the pods. |
| `deployment.hostNetwork` | bool | `nil` | `true` | No | Use host network for the pods. |
| `deployment.hostAliases` | list | `nil` | _(see example)_ | No | Entries injected into the pod's `/etc/hosts`. |
| `deployment.imagePullSecrets` | list | `[]` | `[{name: docker-pull}]` | No | Image pull secrets for private registries. |
| `deployment.command` | list | `[]` | `["/bin/sh"]` | No | Override the container entrypoint command. |
| `deployment.args` | list | `[]` | `["-c", "echo hello"]` | No | Override the container arguments. |
| `deployment.ports` | list | `nil` | `[{containerPort: 8080, name: http}]` | No | Container port definitions. |
| `deployment.env` | object | `nil` | _(see example)_ | No | Environment variables. Supports `value`, `valueFrom.configMapKeyRef`, `valueFrom.secretKeyRef`, etc. |
| `deployment.envFrom` | object | `nil` | _(see example)_ | No | Mount all keys from a ConfigMap or Secret as environment variables. Each entry has `type` (`configmap`/`secret`) and `nameSuffix` or `name`. |
| `deployment.volumes` | object | `nil` | _(see example)_ | No | Volumes to attach to the pod. Key is the volume name; value is the volume spec. Supports Helm templating. |
| `deployment.volumeMounts` | object | `nil` | _(see example)_ | No | Volume mounts for the main container. Key is the volume name; value is the mount spec. |
| `deployment.initContainers` | object | `nil` | _(see example)_ | No | Init containers. Key is the container name; value is the container spec. |
| `deployment.additionalContainers` | list | `nil` | _(see example)_ | No | Sidecar containers appended to the pod spec (no Helm templating). |
| `deployment.resources` | object | `{}` | `{limits: {memory: 256Mi}}` | No | CPU and memory requests/limits for the main container. |
| `deployment.containerSecurityContext` | object | `{readOnlyRootFilesystem: true, runAsNonRoot: true}` | — | No | Security context at the container level. |
| `deployment.securityContext` | object | `{}` | `{fsGroup: 2000}` | No | Security context at the pod level. |
| `deployment.automountServiceAccountToken` | bool | `true` | — | No | Whether to auto-mount the service account token into the pod. |
| `deployment.enableServiceLinks` | bool | `true` | — | No | Whether to inject service environment variables into the pod. |
| `deployment.dnsPolicy` | string | `""` | `ClusterFirst` | No | DNS policy for the pod. |
| `deployment.dnsConfig` | object | `nil` | `{options: [{name: ndots, value: "1"}]}` | No | Custom DNS configuration for the pod. |
| `deployment.terminationGracePeriodSeconds` | int | `nil` | `60` | No | Grace period before the pod is forcefully killed. |
| `deployment.minReadySeconds` | int | `nil` | `10` | No | Minimum seconds a pod must be ready before it is considered available. |
| `deployment.lifecycle` | object | `{}` | `{preStop: {exec: {command: ["sleep","5"]}}}` | No | Lifecycle hooks (`postStart`, `preStop`) for the main container. |
| `deployment.startupProbe.enabled` | bool | `false` | — | No | Enable the startup probe. |
| `deployment.startupProbe.failureThreshold` | int | `30` | — | No | Number of failures before the pod is marked as failed. |
| `deployment.startupProbe.periodSeconds` | int | `10` | — | No | How often (in seconds) to perform the probe. |
| `deployment.startupProbe.successThreshold` | int | `1` | — | No | Minimum consecutive successes to be considered healthy. |
| `deployment.startupProbe.timeoutSeconds` | int | `1` | — | No | Seconds after which the probe times out. |
| `deployment.startupProbe.httpGet` | object | `{}` | `{path: /healthz, port: 8080}` | No | HTTP GET probe definition. |
| `deployment.startupProbe.exec` | object | `{}` | `{command: [cat, /tmp/healthy]}` | No | Exec probe definition. |
| `deployment.startupProbe.tcpSocket` | object | `{}` | `{port: 8080}` | No | TCP socket probe definition. |
| `deployment.startupProbe.grpc` | object | `{}` | `{port: 9090}` | No | gRPC probe definition. |
| `deployment.readinessProbe.enabled` | bool | `false` | — | No | Enable the readiness probe. |
| `deployment.readinessProbe.failureThreshold` | int | `30` | — | No | Number of failures before the pod is marked not ready. |
| `deployment.readinessProbe.periodSeconds` | int | `10` | — | No | How often (in seconds) to perform the probe. |
| `deployment.readinessProbe.successThreshold` | int | `1` | — | No | Minimum consecutive successes to be considered ready. |
| `deployment.readinessProbe.timeoutSeconds` | int | `1` | — | No | Seconds after which the probe times out. |
| `deployment.readinessProbe.httpGet` | object | `{}` | `{path: /ready, port: 8080}` | No | HTTP GET probe definition. |
| `deployment.readinessProbe.exec` | object | `{}` | `{command: [cat, /tmp/ready]}` | No | Exec probe definition. |
| `deployment.readinessProbe.tcpSocket` | object | `{}` | `{port: 8080}` | No | TCP socket probe definition. |
| `deployment.readinessProbe.grpc` | object | `{}` | `{port: 9090}` | No | gRPC probe definition. |
| `deployment.livenessProbe.enabled` | bool | `false` | — | No | Enable the liveness probe. |
| `deployment.livenessProbe.failureThreshold` | int | `30` | — | No | Number of failures before the pod is restarted. |
| `deployment.livenessProbe.periodSeconds` | int | `10` | — | No | How often (in seconds) to perform the probe. |
| `deployment.livenessProbe.successThreshold` | int | `1` | — | No | Minimum consecutive successes to be considered live. |
| `deployment.livenessProbe.timeoutSeconds` | int | `1` | — | No | Seconds after which the probe times out. |
| `deployment.livenessProbe.httpGet` | object | `{}` | `{path: /healthz, port: 8080}` | No | HTTP GET probe definition. |
| `deployment.livenessProbe.exec` | object | `{}` | `{command: [cat, /tmp/healthy]}` | No | Exec probe definition. |
| `deployment.livenessProbe.tcpSocket` | object | `{}` | `{port: 8080}` | No | TCP socket probe definition. |
| `deployment.livenessProbe.grpc` | object | `{}` | `{port: 9090}` | No | gRPC probe definition. |
| `deployment.openshiftOAuthProxy.enabled` | bool | `false` | — | No | Inject an [OpenShift OAuth Proxy](https://github.com/openshift/oauth-proxy) sidecar. OpenShift only. |
| `deployment.openshiftOAuthProxy.port` | int | `8080` | — | No | Port the application listens on (the proxy will forward to this). |
| `deployment.openshiftOAuthProxy.secretName` | string | `openshift-oauth-proxy-tls` | — | No | Name of the Secret holding the TLS certificate for the proxy. |
| `deployment.openshiftOAuthProxy.image` | string | `openshift/oauth-proxy:latest` | — | No | Image for the OAuth Proxy sidecar. |
| `deployment.openshiftOAuthProxy.disableTLSArg` | bool | `false` | — | No | When `true`, the proxy listens on HTTP (`:8081`) instead of HTTPS (`:8443`). Useful when TLS is terminated at the ingress. |

---

### Persistence Parameters

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `persistence.enabled` | bool | `false` | — | No | Create a `PersistentVolumeClaim`. |
| `persistence.mountPVC` | bool | `false` | — | No | Mount the PVC into the Deployment's pods. |
| `persistence.mountPath` | string | `"/"` | `/data` | No | Path inside the container where the PVC is mounted. Only used when `persistence.mountPVC` is `true`. |
| `persistence.name` | string | `""` | `my-app-data` | No | Name of the PVC. Defaults to `<applicationName>-data`. |
| `persistence.accessMode` | string | `ReadWriteOnce` | `ReadWriteMany` | No | Access mode for the PVC. |
| `persistence.storageClass` | string | `null` | `standard` | No | Storage class. `null` = cluster default. `""` or `"-"` = disable dynamic provisioning. |
| `persistence.storageSize` | string | `8Gi` | `20Gi` | No | Requested storage size. |
| `persistence.volumeMode` | string | `""` | `Filesystem` | No | Volume mode (`Filesystem` or `Block`). |
| `persistence.volumeName` | string | `""` | `my-pv` | No | Bind to a specific PersistentVolume by name. |
| `persistence.additionalLabels` | object | `{}` | `env: prod` | No | Extra labels for the PVC. |
| `persistence.annotations` | object | `{}` | `helm.sh/resource-policy: keep` | No | Annotations for the PVC. |

---

### Service Parameters

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `service.enabled` | bool | `true` | — | No | Create a `Service` resource. |
| `service.type` | string | `ClusterIP` | `LoadBalancer` | No | Service type. One of `ClusterIP`, `NodePort`, `LoadBalancer`, `ExternalName`. |
| `service.ports` | list | `[{port: 8080, name: http, protocol: TCP, targetPort: 8080}]` | — | No | List of ports exposed by the service. |
| `service.clusterIP` | string | `nil` | `None` | No | Fixed ClusterIP address. Set to `None` for a headless service. |
| `service.loadBalancerClass` | string | `nil` | `service.k8s.aws/nlb` | No | LoadBalancer class name. Only relevant when `service.type` is `LoadBalancer`. |
| `service.additionalLabels` | object | `nil` | `app.example.com/tier: frontend` | No | Extra labels for the Service. |
| `service.annotations` | object | `nil` | `service.beta.kubernetes.io/aws-load-balancer-type: nlb` | No | Annotations for the Service. |

---

### Ingress Parameters

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `ingress.enabled` | bool | `false` | — | No | Create an `Ingress` resource. |
| `ingress.ingressClassName` | string | `""` | `nginx` | No | Ingress class name (e.g. `nginx`, `traefik`). |
| `ingress.hosts` | list | `[{host: chart-example.local, paths: [{path: /}]}]` | — | No | List of host rules. Each entry has a `host` and a list of `paths`. |
| `ingress.hosts[].host` | string | `chart-example.local` | `myapp.example.com` | No | Hostname for this rule. Supports Helm templating. |
| `ingress.hosts[].paths[].path` | string | `/` | `/api` | No | URL path prefix. |
| `ingress.hosts[].paths[].pathType` | string | `ImplementationSpecific` | `Prefix` | No | Path matching type. One of `Exact`, `Prefix`, `ImplementationSpecific`. |
| `ingress.hosts[].paths[].serviceName` | string | `<applicationName>` | `my-other-svc` | No | Backend service name. Defaults to the application name. |
| `ingress.hosts[].paths[].servicePort` | string | `http` | `8080` | No | Backend service port name or number. |
| `ingress.tls` | list | `nil` | `[{secretName: myapp-tls, hosts: [myapp.example.com]}]` | No | TLS configuration. Secrets must already exist or be created via the `certificate` section. |
| `ingress.additionalLabels` | object | `nil` | `environment: prod` | No | Extra labels for the Ingress. |
| `ingress.annotations` | object | `nil` | `nginx.ingress.kubernetes.io/rewrite-target: /` | No | Annotations for the Ingress. |

---

### HTTPRoute Parameters

> Requires the [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/) CRDs to be installed in the cluster.

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `httpRoute.enabled` | bool | `false` | — | No | Create an `HTTPRoute` (Gateway API) resource. |
| `httpRoute.parentRefs` | list | `nil` | `[{name: my-gateway, sectionName: https}]` | No | Gateway references this route attaches to. Supports Helm templating. |
| `httpRoute.useDefaultGateways` | string | `nil` | `public` | No | Attach to a default Gateway by scope name instead of explicit `parentRefs`. |
| `httpRoute.gatewayNamespace` | string | `""` | `gateway-system` | No | Namespace of the Gateway. Defaults to the same namespace as the HTTPRoute. |
| `httpRoute.hostnames` | list | `nil` | `[myapp.example.com]` | No | Hostnames this route matches. Supports Helm templating. |
| `httpRoute.rules` | list | _(default path prefix `/`)_ | — | No | Routing rules with `matches` and `backendRefs`. Supports Helm templating. |
| `httpRoute.additionalLabels` | object | `{}` | `environment: prod` | No | Extra labels for the HTTPRoute. |
| `httpRoute.annotations` | object | `{}` | `key: value` | No | Annotations for the HTTPRoute. |

---

### Route Parameters (OpenShift)

> Requires an OpenShift cluster.

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `route.enabled` | bool | `false` | — | No | Create an OpenShift `Route` resource. |
| `route.host` | string | `nil` | `myapp.apps.cluster.example.com` | No | Explicit hostname. If empty, OpenShift assigns a default hostname. |
| `route.path` | string | `nil` | `/api` | No | URL path for the route. |
| `route.port.targetPort` | string | `http` | `8443` | No | Target port name or number on the backing service. |
| `route.to.weight` | int | `100` | — | No | Weight for the primary backend. |
| `route.wildcardPolicy` | string | `None` | `Subdomain` | No | Wildcard policy for the route. |
| `route.tls.termination` | string | `edge` | `passthrough` | No | TLS termination strategy. One of `edge`, `reencrypt`, `passthrough`. |
| `route.tls.insecureEdgeTerminationPolicy` | string | `Redirect` | `Allow` | No | How to handle insecure (HTTP) traffic. One of `Allow`, `Redirect`, `None`. |
| `route.alternateBackends` | list | `nil` | `[{kind: Service, name: canary-svc, weight: 20}]` | No | Additional backends with weights for traffic splitting. |
| `route.additionalLabels` | object | `nil` | `environment: prod` | No | Extra labels for the Route. |
| `route.annotations` | object | `nil` | `key: value` | No | Annotations for the Route. |

---

### CronJob Parameters

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `cronJob.enabled` | bool | `false` | — | No | Create `CronJob` resources. |
| `cronJob.jobs` | object | `{}` | _(see example)_ | No | Map of CronJob definitions. The key is used as a name suffix. Each entry supports: `schedule`, `image`, `command`, `args`, `env`, `resources`, `initContainers`, `dnsConfig`, `dnsPolicy`, `enableServiceLinks`, `hostAliases`, `suspend`, `priorityClassName`, `startingDeadlineSeconds`, `activeDeadlineSeconds`. |

Example entry:

```yaml
cronJob:
  enabled: true
  jobs:
    db-cleanup:
      schedule: "0 2 * * *"
      image:
        repository: docker.io/myorg/db-tools
        tag: latest
      command: ["/bin/sh"]
      args: ["-c", "cleanup.sh"]
      resources:
        requests:
          memory: 128Mi
          cpu: 100m
```

---

### Job Parameters

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `job.enabled` | bool | `false` | — | No | Create `Job` resources. |
| `job.jobs` | object | `{}` | _(see example)_ | No | Map of Job definitions. The key is used as a name suffix. Each entry supports: `image`, `command`, `args`, `env`, `resources`, `initContainers`, `dnsConfig`, `dnsPolicy`, `enableServiceLinks`, `hostAliases`, `activeDeadlineSeconds`, `additionalPodAnnotations` (useful for Helm hooks). |

Example entry:

```yaml
job:
  enabled: true
  jobs:
    db-migration:
      additionalPodAnnotations:
        helm.sh/hook: pre-install,pre-upgrade
        helm.sh/hook-weight: "-1"
        helm.sh/hook-delete-policy: before-hook-creation
      image:
        repository: docker.io/myorg/migrator
        tag: latest
      command: ["/bin/sh", "-c", "migrate.sh"]
```

---

### RBAC Parameters

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `rbac.enabled` | bool | `true` | — | No | Enable RBAC resource creation (Roles, RoleBindings). |
| `rbac.serviceAccount.enabled` | bool | `false` | — | No | Create a `ServiceAccount` for the application. |
| `rbac.serviceAccount.name` | string | `""` | `my-app-sa` | No | Name of the ServiceAccount. Defaults to the application name. |
| `rbac.serviceAccount.additionalLabels` | object | `{}` | `team: backend` | No | Extra labels for the ServiceAccount. |
| `rbac.serviceAccount.annotations` | object | `{}` | `eks.amazonaws.com/role-arn: arn:aws:iam::123:role/my-role` | No | Annotations for the ServiceAccount. Commonly used for IRSA (AWS) or Workload Identity (GCP). |
| `rbac.roles` | list | `nil` | _(see example)_ | No | List of namespaced `Role` definitions to create and bind to the ServiceAccount. Each entry has `name` and `rules`. |

Example entry:

```yaml
rbac:
  enabled: true
  serviceAccount:
    enabled: true
  roles:
    - name: configmap-reader
      rules:
        - apiGroups: [""]
          resources: ["configmaps"]
          verbs: ["get", "list"]
```

---

### ConfigMap Parameters

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `configMap.enabled` | bool | `false` | — | No | Create `ConfigMap` resources. |
| `configMap.additionalLabels` | object | `{}` | `app: my-app` | No | Extra labels applied to all ConfigMaps. |
| `configMap.annotations` | object | `{}` | `key: value` | No | Annotations applied to all ConfigMaps. |
| `configMap.files` | object | `nil` | _(see example)_ | No | Map of ConfigMap entries. The key is used as a name suffix; the value is the ConfigMap data (key-value pairs). |

Example entry:

```yaml
configMap:
  enabled: true
  files:
    app-config:
      LOG_LEVEL: info
      FEATURE_FLAG: "true"
```

---

### Secret Parameters

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `secret.enabled` | bool | `false` | — | No | Create `Secret` resources. |
| `secret.additionalLabels` | object | `{}` | `app: my-app` | No | Extra labels applied to all Secrets. |
| `secret.annotations` | object | `{}` | `key: value` | No | Annotations applied to all Secrets. |
| `secret.files` | object | `nil` | _(see example)_ | No | Map of Secret entries. The key is used as a name suffix. Three modes are supported: `data` (chart base64-encodes the values), `encodedData` (values are already base64-encoded), `stringData` (raw string values). |

Example entry:

```yaml
secret:
  enabled: true
  files:
    db-credentials:
      data:
        DB_PASSWORD: mysecretpassword
    api-key:
      stringData:
        API_KEY: my-raw-api-key
```

---

### SealedSecret Parameters

> Requires the [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) controller to be installed in the cluster.

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `sealedSecret.enabled` | bool | `false` | — | No | Create `SealedSecret` resources. |
| `sealedSecret.additionalLabels` | object | `{}` | `app: my-app` | No | Extra labels applied to all SealedSecrets. |
| `sealedSecret.annotations` | object | `{}` | `key: value` | No | Annotations applied to all SealedSecrets. |
| `sealedSecret.files` | object | `nil` | _(see example)_ | No | Map of SealedSecret entries. The key is used as a name suffix. Each entry has `encryptedData` (map of encrypted values), optional `annotations`, `labels`, and `clusterWide` (bool). |

---

### ExternalSecret Parameters

> Requires the [External Secrets Operator](https://external-secrets.io) to be installed in the cluster.

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `externalSecret.enabled` | bool | `false` | — | No | Create `ExternalSecret` resources. |
| `externalSecret.additionalLabels` | object | `nil` | `app: my-app` | No | Extra labels applied to all ExternalSecrets. |
| `externalSecret.annotations` | object | `nil` | `key: value` | No | Annotations applied to all ExternalSecrets. |
| `externalSecret.secretStore.name` | string | `tenant-vault-secret-store` | `my-store` | No | Default SecretStore name. Can be overridden per entry in `files`. |
| `externalSecret.secretStore.kind` | string | `SecretStore` | `ClusterSecretStore` | No | Kind of the SecretStore. Either `SecretStore` or `ClusterSecretStore`. |
| `externalSecret.refreshInterval` | string | `1m` | `5m` | No | How often to re-fetch secrets from the provider. |
| `externalSecret.files` | object | `nil` | _(see example)_ | No | Map of ExternalSecret entries. The key is used as a name suffix. Each entry uses either `data` (explicit key mapping) or `dataFrom` (fetch all keys from a path). |

Example entry:

```yaml
externalSecret:
  enabled: true
  files:
    db-secret:
      data:
        db-password:
          remoteRef:
            key: myapp/db
            property: password
```

---

### SecretProviderClass Parameters

> Requires the [Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/) to be installed in the cluster.

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `secretProviderClass.enabled` | bool | `false` | — | No | Create a `SecretProviderClass` resource. |
| `secretProviderClass.name` | string | `""` | `my-spc` | No | Name of the SecretProviderClass. Required when enabled. |
| `secretProviderClass.provider` | string | `""` | `vault` | No | Provider name (e.g. `vault`, `aws`, `azure`, `gcp`). Required when enabled. |
| `secretProviderClass.vaultAddress` | string | `""` | `http://vault:8200` | No | Vault server address. Required when `provider` is `vault`. |
| `secretProviderClass.roleName` | string | `""` | `my-role` | No | Vault role name. Required when `provider` is `vault`. Supports Helm templating. |
| `secretProviderClass.objects` | list | `nil` | _(see example)_ | No | List of secret object definitions to fetch from the provider. |
| `secretProviderClass.secretObjects` | list | `nil` | _(see example)_ | No | Mapping of provider secrets to Kubernetes Secret keys. |

---

### ServiceMonitor Parameters

> Requires the [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator) to be installed in the cluster.

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `serviceMonitor.enabled` | bool | `false` | — | No | Create a `ServiceMonitor` resource for Prometheus scraping. |
| `serviceMonitor.additionalLabels` | object | `{}` | `release: prometheus` | No | Extra labels for the ServiceMonitor. Used by Prometheus to discover it. |
| `serviceMonitor.annotations` | object | `{}` | `key: value` | No | Annotations for the ServiceMonitor. |
| `serviceMonitor.endpoints` | list | `[{interval: 5s, path: /actuator/prometheus, port: http}]` | — | No | List of scrape endpoints. Each entry supports standard Prometheus endpoint fields (`path`, `port`, `interval`, `scheme`, etc.). |

---

### AlertmanagerConfig Parameters

> Requires the [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator) to be installed in the cluster.

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `alertmanagerConfig.enabled` | bool | `false` | — | No | Create an `AlertmanagerConfig` resource. |
| `alertmanagerConfig.selectionLabels` | object | `{alertmanagerConfig: workload}` | — | No | Labels used by Alertmanager to select this config and merge it into the base configuration. |
| `alertmanagerConfig.spec.route` | object | `nil` | _(see example)_ | No | Top-level alert routing rule for this namespace. |
| `alertmanagerConfig.spec.receivers` | list | `[]` | _(see example)_ | No | List of alert receivers (e.g. Slack, PagerDuty). |
| `alertmanagerConfig.spec.inhibitRules` | list | `[]` | _(see example)_ | No | Inhibition rules to suppress lower-severity alerts when a higher-severity alert is firing. |

---

### PrometheusRule Parameters

> Requires the [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator) to be installed in the cluster.

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `prometheusRule.enabled` | bool | `false` | — | No | Create a `PrometheusRule` resource. |
| `prometheusRule.additionalLabels` | object | `{}` | `prometheus: kube-prometheus` | No | Extra labels for the PrometheusRule. Used by Prometheus to discover it. |
| `prometheusRule.groups` | list | `[]` | _(see example)_ | No | List of rule groups. Each group has a `name` and a list of `rules` (alert or recording rules). |

---

### Autoscaling (HPA) Parameters

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `autoscaling.enabled` | bool | `false` | — | No | Create a `HorizontalPodAutoscaler` resource. When enabled, consider leaving `deployment.replicas` unset. |
| `autoscaling.minReplicas` | int | `1` | — | No | Minimum number of replicas. |
| `autoscaling.maxReplicas` | int | `10` | — | No | Maximum number of replicas. |
| `autoscaling.metrics` | list | _(CPU + memory at 60%)_ | — | No | List of metrics used to drive scaling decisions. Supports any HPA v2 metric type. |
| `autoscaling.additionalLabels` | object | `{}` | `team: backend` | No | Extra labels for the HPA. |
| `autoscaling.annotations` | object | `{}` | `key: value` | No | Annotations for the HPA. |

---

### VPA Parameters

> Requires the [Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) to be installed in the cluster.

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `vpa.enabled` | bool | `false` | — | No | Create a `VerticalPodAutoscaler` resource. |
| `vpa.updatePolicy.updateMode` | string | `Auto` | `Off` | No | VPA update mode. One of `Auto`, `Recreate`, `Initial`, `Off`. |
| `vpa.containerPolicies` | list | `[]` | _(see example)_ | No | Per-container resource policies (min/max allowed, controlled resources). |
| `vpa.additionalLabels` | object | `{}` | `team: backend` | No | Extra labels for the VPA. |
| `vpa.annotations` | object | `{}` | `key: value` | No | Annotations for the VPA. |

---

### PodDisruptionBudget Parameters

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `pdb.enabled` | bool | `false` | — | No | Create a `PodDisruptionBudget` resource. |
| `pdb.minAvailable` | int | `1` | — | No | Minimum number of pods that must remain available during voluntary disruptions. Mutually exclusive with `maxUnavailable`. |
| `pdb.maxUnavailable` | int | `nil` | `1` | No | Maximum number of pods that may be unavailable during voluntary disruptions. Mutually exclusive with `minAvailable`. |

---

### NetworkPolicy Parameters

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `networkPolicy.enabled` | bool | `false` | — | No | Create a `NetworkPolicy` resource. |
| `networkPolicy.ingress` | list | `nil` | _(see example)_ | No | Ingress rules. Each rule specifies allowed sources (`from`) and ports. |
| `networkPolicy.egress` | list | `nil` | _(see example)_ | No | Egress rules. Each rule specifies allowed destinations (`to`) and ports. |
| `networkPolicy.additionalLabels` | object | `nil` | `env: prod` | No | Extra labels for the NetworkPolicy. |
| `networkPolicy.annotations` | object | `nil` | `key: value` | No | Annotations for the NetworkPolicy. |

---

### Certificate Parameters

> Requires [cert-manager](https://cert-manager.io) to be installed in the cluster.

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `certificate.enabled` | bool | `false` | — | No | Create a cert-manager `Certificate` resource. |
| `certificate.secretName` | string | `tls-cert` | `myapp-tls` | No | Name of the Kubernetes Secret that cert-manager will create and populate with the certificate. |
| `certificate.duration` | string | `8760h0m0s` | `2160h0m0s` | No | Requested certificate lifetime (1 year by default). |
| `certificate.renewBefore` | string | `720h0m0s` | `360h0m0s` | No | How long before expiry cert-manager starts renewing (30 days by default). |
| `certificate.subject` | object | `nil` | `{organizations: [myorg]}` | No | Full X.509 subject fields (organization, country, etc.). |
| `certificate.commonName` | string | `nil` | `myapp.example.com` | No | Certificate common name. Not recommended for end-entity certificates; prefer `dnsNames`. |
| `certificate.keyAlgorithm` | string | `RSA` | `ECDSA` | No | Private key algorithm. |
| `certificate.keyEncoding` | string | `PKCS1` | `PKCS8` | No | Private key encoding standard. |
| `certificate.keySize` | int | `2048` | `4096` | No | Key size in bits. |
| `certificate.isCA` | bool | `false` | — | No | Mark this certificate as a CA certificate. |
| `certificate.usages` | list | `nil` | `[digital signature, client auth]` | No | X.509 key usages. |
| `certificate.dnsNames` | list | `nil` | `[myapp.example.com]` | No | DNS Subject Alternative Names. |
| `certificate.ipAddresses` | list | `nil` | `[192.168.0.5]` | No | IP address Subject Alternative Names. |
| `certificate.uriSANs` | list | `nil` | `[spiffe://cluster.local/ns/default/sa/myapp]` | No | URI Subject Alternative Names. |
| `certificate.emailSANs` | list | `nil` | `[admin@example.com]` | No | Email Subject Alternative Names. |
| `certificate.privateKey.enabled` | bool | `false` | — | No | Enable private key configuration. |
| `certificate.privateKey.rotationPolicy` | string | `Always` | `Never` | No | When to rotate the private key on renewal. |
| `certificate.issuerRef.name` | string | `ca-issuer` | `letsencrypt-prod` | No | Name of the cert-manager Issuer or ClusterIssuer. |
| `certificate.issuerRef.kind` | string | `ClusterIssuer` | `Issuer` | No | Kind of the issuer. |
| `certificate.issuerRef.group` | string | `cert-manager.io` | — | No | API group of the issuer. |
| `certificate.keystores.enabled` | bool | `false` | — | No | Enable keystore output formats (PKCS12, JKS) stored in the certificate Secret. |
| `certificate.keystores.pkcs12.create` | bool | `true` | — | No | Create a PKCS12 keystore entry in the Secret. |
| `certificate.keystores.pkcs12.key` | string | `test_key` | `keystore.p12` | No | Key name in the Secret for the PKCS12 data. |
| `certificate.keystores.pkcs12.name` | string | `test-creds` | `my-keystore-secret` | No | Name of the Secret containing the PKCS12 password. |
| `certificate.keystores.jks.create` | bool | `false` | — | No | Create a JKS keystore entry in the Secret. |
| `certificate.keystores.jks.key` | string | `test_key` | `keystore.jks` | No | Key name in the Secret for the JKS data. |
| `certificate.keystores.jks.name` | string | `test-creds` | `my-keystore-secret` | No | Name of the Secret containing the JKS password. |
| `certificate.additionalLabels` | object | `{}` | `env: prod` | No | Extra labels for the Certificate. |
| `certificate.annotations` | object | `{}` | `key: value` | No | Annotations for the Certificate. |

---

### GrafanaDashboard Parameters

> Requires the [Grafana Operator](https://github.com/grafana/grafana-operator) to be installed in the cluster.

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `grafanaDashboard.enabled` | bool | `false` | — | No | Create `GrafanaDashboard` resources. |
| `grafanaDashboard.contents` | object | `nil` | _(see example)_ | No | Map of dashboard entries. The key is used as a name suffix. Each entry supports `json` (inline JSON), `url` (remote JSON file), or `configMapRef`. The `url` field takes precedence over `json`. |
| `grafanaDashboard.additionalLabels` | object | `{}` | `grafanaDashboard: grafana-operator` | No | Extra labels for all GrafanaDashboard resources. |
| `grafanaDashboard.annotations` | object | `{}` | `key: value` | No | Annotations for all GrafanaDashboard resources. |

---

### EndpointMonitor Parameters

> Requires the [IngressMonitorController](https://github.com/stakater/IngressMonitorController) to be installed in the cluster.

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `endpointMonitor.enabled` | bool | `false` | — | No | Create an `EndpointMonitor` resource for uptime monitoring. |
| `endpointMonitor.additionalLabels` | object | `{}` | `team: ops` | No | Extra labels for the EndpointMonitor. |
| `endpointMonitor.annotations` | object | `{}` | `key: value` | No | Annotations for the EndpointMonitor. |

---

### Forecastle Parameters

> Requires [Forecastle](https://github.com/stakater/Forecastle) to be installed in the cluster.

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `forecastle.enabled` | bool | `false` | — | No | Create a `ForecastleApp` resource to expose the application in the Forecastle dashboard. |
| `forecastle.displayName` | string | `""` | `My App` | No | Display name shown in the Forecastle dashboard. Required when enabled. |
| `forecastle.icon` | string | _(stakater icon URL)_ | `https://example.com/icon.png` | No | URL of the application icon shown in the dashboard. |
| `forecastle.group` | string | `""` | `Platform Tools` | No | Group/category in the dashboard. Defaults to the release namespace. |
| `forecastle.networkRestricted` | bool | `false` | — | No | Mark the application as network-restricted in the dashboard. |
| `forecastle.properties` | object | `nil` | `{Owner: team-a}` | No | Custom key-value properties shown in the dashboard entry. |
| `forecastle.additionalLabels` | object | `nil` | `env: prod` | No | Extra labels for the ForecastleApp. |

---

### Backup Parameters

> Requires [Velero](https://velero.io) or [OADP](https://github.com/openshift/oadp-operator) to be installed in the cluster.

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `backup.enabled` | bool | `false` | — | No | Create a Velero `Backup` resource. |
| `backup.namespace` | string | _(release namespace)_ | `my-namespace` | No | Namespace in which the Backup resource is created. |
| `backup.defaultVolumesToRestic` | bool | `true` | — | No | Use Restic to snapshot all pod volumes by default. |
| `backup.snapshotVolumes` | bool | `true` | — | No | Take cloud-provider snapshots of PersistentVolumes. |
| `backup.storageLocation` | string | `nil` | `default` | No | Name of the Velero `BackupStorageLocation` to use. |
| `backup.ttl` | string | `1h0m0s` | `720h0m0s` | No | How long to retain the backup before deletion. |
| `backup.includedNamespaces` | list | _(release namespace)_ | `[production]` | No | Namespaces whose resources are included in the backup. |
| `backup.includedResources` | list | `nil` | `[deployments, services]` | No | Resource types to include. Empty means all types. |
| `backup.excludedResources` | list | `nil` | `[events]` | No | Resource types to exclude from the backup. |
| `backup.additionalLabels` | object | `{}` | `env: prod` | No | Extra labels for the Backup resource. |
| `backup.annotations` | object | `{}` | `key: value` | No | Annotations for the Backup resource. |

---

### Extra Objects

`extraObjects` lets you render arbitrary Kubernetes manifests as part of the same Helm release. Each entry is passed through `tpl`, so you can use Helm templating inside them. The value can be a list or a map (keys are ignored when it is a map).

| Key | Type | Default | Example | Required | Description |
|-----|------|---------|---------|----------|-------------|
| `extraObjects` | list/object | `[]` | _(see example)_ | No | Arbitrary Kubernetes manifests rendered alongside the chart's own resources. Supports Helm templating. |

Example:

```yaml
extraObjects:
  - apiVersion: v1
    kind: ConfigMap
    metadata:
      name: '{{ .Release.Name }}-extra-config'
      namespace: '{{ .Release.Namespace }}'
    data:
      EXTRA_KEY: extra-value
  - |
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: '{{ .Release.Name }}-extra-sa'
```
