# OpenShift Service Mesh 3.4.3: Sidecar Observability and Tracing

Reusable setup guide — reconciled with the working ROSA installation on 24 October 2026.

This guide supersedes the earlier draft runbook. It starts with an installed, healthy Service Mesh 3 sidecar control plane and adds metrics, Kiali, Tempo with AWS S3, an OpenTelemetry Collector, direct gateway ingress, and the OpenShift trace waterfall viewer.

The data paths and fixes were verified during our session. This consolidated procedure also improves installation order, service-account ownership, and monitor target selection. It has not been run as a fresh installation on a second cluster. Use the checkpoints: an accepted manifest does not prove end-to-end delivery.

## Architecture and Baseline

```text
Browser → AWS load balancer → Istio ingress gateway → Productpage
                                                      ├─ Details
                                                      └─ Reviews → Ratings (v2/v3)

Envoy metrics → user-workload Prometheus → Thanos Querier → Kiali graph

Envoy spans → OpenTelemetry Collector → authenticated Tempo gateway
                                       → Tempo → AWS S3

Tempo gateway → Kiali workload traces
             → OpenShift Observe → Traces → waterfall
```

Gateway API configures the gateway proxy; it is not another traffic-processing hop. The direct Bookinfo ingress path does not require an OpenShift Route.

| Component | Baseline |
|---|---|
| Service Mesh Operator | 3.4.3 |
| Istio and IstioCNI | v1.30.5 |
| Kiali Operator and server | 2.27.5 |
| OpenShift supported by this mesh release | 4.20 or later |
| Tempo Operator observed in our session | 0.22.0-4 |
| Tempo observed in our session | 2.10.8 |
| Red Hat OpenTelemetry Collector observed | 0.158.0 |

The mesh compatibility values are from the [Red Hat component table](https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.4/html/release_notes/ossm-release-notes-version-support-tables). The observed Tempo and Collector versions record this environment, not compatibility guarantees for future releases.

### Names Used Consistently

| Purpose | Name |
|---|---|
| Istio resource / active revision | default |
| Control-plane namespace | istio-system |
| Application and collector namespace | bookinfo |
| Tempo namespace / TempoStack | tempo-system / tempo |
| Tenant | dev |
| Kiali resource / service account | kiali / kiali-service-account |
| Collector resource / generated Service | otel / otel-collector |
| Explicit collector service account | otel-tempo-writer |
| Gateway / HTTPRoute | bookinfo-gateway / bookinfo |

Keep these names for the first installation. If changing them, update every matching reference, including FQDNs and RBAC subjects.

The baseline deliberately keeps the collector in Bookinfo, matching the working topology. It can receive spans from other namespaces. Section 15 explains centralized deployment.

---

## 1. Prerequisites and Existing Mesh

Run the commands from Bash on your bastion. Required tools: oc, AWS CLI v2, jq, curl, and a DNS lookup utility such as getent.

Your OpenShift identity needs operator installation, monitoring configuration, and cluster RBAC privileges. The AWS provisioning identity needs S3 permissions and, for the dedicated credential procedure below, IAM user/policy/key permissions.

```bash
oc whoami
oc version
oc get clusterversion version
aws sts get-caller-identity
aws configure get region
mkdir -p work

oc get istio default -o yaml
oc get istiocni
oc get istiorevision
oc get pods -n istio-system
oc get namespaces -L istio.io/rev,istio-injection,istio.io/dataplane-mode
```

Checkpoint: Istio/default is healthy, targets istio-system through spec.namespace, uses v1.30.5, and has active revision default. Istio custom resources are cluster-scoped; their spec.namespace identifies the deployed control plane.

This guide assumes sidecar mode. Do not enroll Bookinfo in ambient mode. A gateway pod can correctly be 1/1; a meshed Bookinfo application pod should be 2/2. With native sidecars, istio-proxy may appear as a restartable init container.

If the mesh itself is missing, finish the [Service Mesh installation procedure](https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.4/html/installing/ossm-installing-service-mesh) first. Do not recreate or replace a healthy control plane just to follow this guide.

---

## 2. Install the Supporting Red Hat Operators

Inspect existing installations first:

```bash
oc get csv -A | grep -Ei 'kiali|tempo|opentelemetry|observability'
```

OLM copies CSVs into namespaces. Repeated CSV rows do not necessarily mean separate operator deployments.

For missing operators, open Ecosystem → Software Catalog on OpenShift 4.20+, or Operators → OperatorHub on older consoles. Install the Red Hat offerings:

| Operator | Purpose |
|---|---|
| Kiali Operator provided by Red Hat | Mesh UI |
| Tempo Operator | TempoStack and trace storage |
| Red Hat build of OpenTelemetry | OpenTelemetryCollector |
| Cluster Observability Operator | Distributed tracing UI plugin |

For Cluster Observability Operator, use stable, all namespaces, the recommended openshift-cluster-observability-operator namespace, and its recommended namespace-monitoring option. Choose approval settings according to your upgrade policy. For the other operators use supported catalog settings and versions compatible with the baseline; do not invent Subscription package/channel names.

Wait for Succeeded. Verify the actual CRDs:

```bash
oc get crd kialis.kiali.io \
  tempostacks.tempo.grafana.com \
  opentelemetrycollectors.opentelemetry.io \
  uiplugins.observability.openshift.io \
  gateways.gateway.networking.k8s.io \
  httproutes.gateway.networking.k8s.io
```

Gateway API CRDs are platform-managed on OpenShift 4.19+. Do not install community CRDs over them.

References: [COO installation](https://docs.redhat.com/en/documentation/red_hat_openshift_cluster_observability_operator/1-latest/observability/installing_red_hat_openshift_cluster_observability_operator/configuring-the-cluster-observability-operator-to-monitor-a-service-1), [Service Mesh gateway deployment](https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.4/html/installing/ossm-installing-service-mesh).

---

## 3. Enable User-Workload Monitoring

```bash
oc get configmap cluster-monitoring-config -n openshift-monitoring -o yaml
oc get pods -n openshift-user-workload-monitoring
```

If enableUserWorkload is already true and monitoring pods are ready, skip the configuration change.

If the ConfigMap does not exist, create it:

```bash
oc apply -f - <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
EOF
```

If it already contains other settings, back it up and edit its embedded config.yaml. Preserve those settings and add/change the top-level enableUserWorkload entry:

```bash
oc get configmap cluster-monitoring-config -n openshift-monitoring \
  -o yaml > work/cluster-monitoring-config.before.yaml
oc edit configmap cluster-monitoring-config -n openshift-monitoring
```

Do not apply the minimal example over an existing ConfigMap containing other monitoring settings.

Checkpoint:

```bash
oc get pods -n openshift-user-workload-monitoring
```

Wait for the user-workload Prometheus, operator, and Thanos Ruler pods to be ready.

---

## 4. Create Namespaces and Enable Sidecars

```bash
oc create namespace bookinfo --dry-run=client -o yaml | oc apply -f -
oc create namespace tempo-system --dry-run=client -o yaml | oc apply -f -
oc label namespace bookinfo istio.io/rev=default istio-discovery=enabled --overwrite
oc get namespace bookinfo --show-labels
```

For a reused namespace, choose one injection mechanism. This guide uses the revision label. If the legacy istio-injection label exists, remove it before deploying applications:

```bash
# Only if that legacy label is present:
# oc label namespace bookinfo istio-injection-
```

If your Istio has restrictive discovery selectors, ensure it discovers Bookinfo and the collector namespace. The session's baseline did not have restrictive selectors.

---

## 5. Create AWS S3 Storage and Credentials

### 5.1 Set Your AWS Values

Edit the region and deployment identifier. Keep this shell open until Secret creation.

```bash
export AWS_REGION=us-east-2
export TEMPO_DEPLOYMENT=mesh-lab
export TEMPO_AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export TEMPO_BUCKET="tempo-traces-$TEMPO_AWS_ACCOUNT_ID-$AWS_REGION-$TEMPO_DEPLOYMENT"
export TEMPO_IAM_USER="tempo-s3-$TEMPO_DEPLOYMENT"
printf 'Region: %s\nBucket: %s\n' "$AWS_REGION" "$TEMPO_BUCKET"
```

Use a unique bucket per independent Tempo installation. These endpoint/ARN examples target standard commercial AWS regions; adjust for other AWS partitions.

### 5.2 Create the Private Bucket Once

If reusing a bucket you own, verify it with head-bucket and skip creation. A 403 is an authorization problem, not proof that a bucket is absent.

```bash
if [ "$AWS_REGION" = "us-east-1" ]; then
  aws s3api create-bucket --bucket "$TEMPO_BUCKET" --region "$AWS_REGION"
else
  aws s3api create-bucket --bucket "$TEMPO_BUCKET" --region "$AWS_REGION" \
    --create-bucket-configuration "LocationConstraint=$AWS_REGION"
fi

aws s3api put-public-access-block --bucket "$TEMPO_BUCKET" \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

aws s3api put-bucket-encryption --bucket "$TEMPO_BUCKET" \
  --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'

aws s3api head-bucket --bucket "$TEMPO_BUCKET" --region "$AWS_REGION"
```

Depending on AWS CLI version, head-bucket can succeed without output. [AWS bucket creation reference](https://docs.aws.amazon.com/cli/latest/reference/s3api/create-bucket.html).

### 5.3 Create a Bucket-Scoped IAM Identity

The session used static credentials. This guide uses a dedicated IAM user instead of copying a human administrator's broad credentials. For production ROSA, consider supported STS/workload identity; it uses a different Secret format.

For a new installation, create the IAM user once:

```bash
aws iam create-user --user-name "$TEMPO_IAM_USER"
```

If it exists, inspect it with aws iam get-user. Do not recreate it or generate access keys on each rerun.

Create and attach the bucket-scoped policy:

```bash
jq -n --arg bucket "$TEMPO_BUCKET" '{
  Version: "2012-10-17",
  Statement: [
    {
      Effect: "Allow",
      Action: ["s3:ListBucket", "s3:GetBucketLocation", "s3:ListBucketMultipartUploads"],
      Resource: ("arn:aws:s3:::" + $bucket)
    },
    {
      Effect: "Allow",
      Action: ["s3:GetObject", "s3:PutObject", "s3:DeleteObject", "s3:AbortMultipartUpload", "s3:ListMultipartUploadParts"],
      Resource: ("arn:aws:s3:::" + $bucket + "/*")
    }
  ]
}' > work/tempo-s3-policy.json

aws iam put-user-policy --user-name "$TEMPO_IAM_USER" \
  --policy-name tempo-bucket-access \
  --policy-document file://work/tempo-s3-policy.json
```

For a NEW key, capture its values without printing them. Run once, immediately followed by Secret creation:

```bash
TEMPO_KEY_JSON=$(aws iam create-access-key --user-name "$TEMPO_IAM_USER" --output json)
TEMPO_ACCESS_KEY_ID=$(printf '%s' "$TEMPO_KEY_JSON" | jq -er '.AccessKey.AccessKeyId')
TEMPO_ACCESS_KEY_SECRET=$(printf '%s' "$TEMPO_KEY_JSON" | jq -er '.AccessKey.SecretAccessKey')
test -n "$TEMPO_ACCESS_KEY_ID" && test -n "$TEMPO_ACCESS_KEY_SECRET"
```

If IAM provisioning belongs to your AWS administrator, obtain a dedicated credential through your approved secret process instead, and populate those two variables. Do not use temporary/session credentials with this four-field static-key recipe.

### 5.4 Create the S3 Secret

```bash
oc create secret generic tempo-s3 -n tempo-system \
  --from-literal=bucket="$TEMPO_BUCKET" \
  --from-literal=endpoint="https://s3.$AWS_REGION.amazonaws.com" \
  --from-literal=access_key_id="$TEMPO_ACCESS_KEY_ID" \
  --from-literal=access_key_secret="$TEMPO_ACCESS_KEY_SECRET" \
  --dry-run=client -o yaml | oc apply -f -

unset TEMPO_KEY_JSON TEMPO_ACCESS_KEY_ID TEMPO_ACCESS_KEY_SECRET

oc get secret tempo-s3 -n tempo-system -o json | jq '.data | keys'
```

Expected keys: access_key_id, access_key_secret, bucket, endpoint.

**Corrections from the session:**
- Include `https://` in the endpoint.
- Do not include region in this static-key Secret for the observed Tempo Operator. It rejected that mixture as long-lived and short-lived configuration.
- STS uses fields such as region and role_arn instead; it is a separate recipe.
- Do not print Secret values or enable shell tracing while handling credentials.

References: [Tempo static credentials and STS formats](https://grafana.com/docs/tempo/latest/set-up-for-tracing/setup-tempo/deploy/kubernetes/operator/object-storage/), [AWS S3 policies](https://docs.aws.amazon.com/AmazonS3/latest/userguide/example-policies-s3.html).

---

## 6. Create TempoStack with Its Gateway and Tenant

Check for a default StorageClass:

```bash
oc get storageclass
```

If none is default, select a supported class and add spec.storageClassName to the manifest before applying it.

```bash
oc apply -f - <<'EOF'
apiVersion: tempo.grafana.com/v1alpha1
kind: TempoStack
metadata:
  name: tempo
  namespace: tempo-system
spec:
  storage:
    secret:
      name: tempo-s3
      type: s3
  storageSize: 10Gi
  tenants:
    mode: openshift
    authentication:
      - tenantName: dev
        tenantId: "1610b0c3-c509-4592-a256-a1871353dbfa"
  template:
    gateway:
      enabled: true
      rbac:
        enabled: true
    queryFrontend:
      jaegerQuery:
        enabled: false
EOF

oc wait --for=condition=Ready tempostack/tempo -n tempo-system --timeout=600s
oc get tempostack tempo -n tempo-system
oc get pods,pvc,svc,route -n tempo-system
```

Checkpoint: TempoStack READY=True; PVCs bind; all pods ready; gateway normally 2/2. This is the modest initial configuration used for learning, not an HA/capacity specification.

`dev` is the gateway-facing tenant name; the UUID is its internal identifier. Keep them consistent. Tenant permissions are global across stacks sharing that tenant name; use distinct tenants when isolating other installations in the same cluster.

No manually discovered OAuth issuer or OIDC client Secret is required for this openshift tenancy configuration.

---

## 7. Create Collector and Viewer Permissions First

Use an explicit service account whose name differs from the operator-generated otel-collector account. The session encountered token mount failures when that generated account disappeared. Keeping an independent account is a consolidation improvement.

```bash
oc apply -f - <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: otel-tempo-writer
  namespace: bookinfo
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: tempo-dev-traces-writer
rules:
  - apiGroups: [tempo.grafana.com]
    resources: [dev]
    resourceNames: [traces]
    verbs: [create]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tempo-dev-collector-writer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: tempo-dev-traces-writer
subjects:
  - kind: ServiceAccount
    name: otel-tempo-writer
    namespace: bookinfo
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: tempo-dev-traces-reader
rules:
  - apiGroups: [tempo.grafana.com]
    resources: [dev]
    resourceNames: [traces]
    verbs: [get]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kiali-tempo-dev-reader
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: tempo-dev-traces-reader
subjects:
  - kind: ServiceAccount
    name: kiali-service-account
    namespace: istio-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kiali-bookinfo-trace-attributes
  namespace: bookinfo
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
  - kind: ServiceAccount
    name: kiali-service-account
    namespace: istio-system
EOF

oc create clusterrolebinding tempo-dev-console-user \
  --clusterrole=tempo-dev-traces-reader --user="$(oc whoami)" \
  --dry-run=client -o yaml | oc apply -f -
```

The last command grants the current console user tenant read access. A non-admin user also needs the intended namespace's view permissions to see its attributes. Grant access to specific users/projects rather than all authenticated users.

Tempo distinguishes tenant read permission from namespace-level attribute access. The tenant/traces resource is virtual; `oc auth can-i create dev/traces` without the API group was an incorrect test. Use an actual authenticated trace API query as the final verification.

References: [Tempo authenticated collector export and tenancy](https://docs.redhat.com/en/documentation/red_hat_openshift_distributed_tracing_platform/3.10/html/installing_distributed_tracing/distr-tracing-tempo-installing), [query RBAC](https://docs.redhat.com/en/documentation/red_hat_openshift_distributed_tracing_platform/3.9/html/configuring_distributed_tracing/distr-tracing-tempo-configuring).

---

## 8. Deploy the Collector with Namespace Enrichment

This collector receives a sidecar because Bookinfo is enrolled for injection, matching the working session. Do not independently disable it in a STRICT-mTLS mesh without adapting receiver/mesh policies.

The transform processor derives resource k8s.namespace.name from each span's istio.namespace. It does not hard-code all traffic to Bookinfo, so it also works for spans from other Istio namespaces.

```bash
oc apply -f - <<'EOF'
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otel
  namespace: bookinfo
spec:
  mode: deployment
  serviceAccount: otel-tempo-writer
  config:
    extensions:
      bearertokenauth:
        filename: /var/run/secrets/kubernetes.io/serviceaccount/token
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    processors:
      transform/namespace:
        error_mode: propagate
        trace_statements:
          - context: span
            statements:
              - set(resource.attributes["k8s.namespace.name"], attributes["istio.namespace"]) where resource.attributes["k8s.namespace.name"] == nil and attributes["istio.namespace"] != nil
      batch: {}
    exporters:
      otlp/dev:
        endpoint: tempo-tempo-gateway.tempo-system.svc.cluster.local:8090
        tls:
          insecure: false
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/service-ca.crt
        auth:
          authenticator: bearertokenauth
        headers:
          X-Scope-OrgID: dev
    service:
      extensions: [bearertokenauth]
      pipelines:
        traces:
          receivers: [otlp]
          processors: [transform/namespace, batch]
          exporters: [otlp/dev]
EOF

oc rollout status deployment/otel-collector -n bookinfo --timeout=180s
oc get pods,svc -n bookinfo -l app.kubernetes.io/instance=bookinfo.otel
oc logs -n bookinfo deployment/otel-collector -c otc-container --tail=60
```

Checkpoint: collector is ready; Service otel-collector exposes 4317/4318. Its instance label is `bookinfo.otel`, not `otel`.

The observed Collector 0.158.0 warns that the otlp exporter alias is deprecated in favor of otlp_grpc. The alias worked in this session and is retained for reproducibility. If adopting the new spelling, change both the exporter identifier and its pipeline reference after checking support in your installed build.

Do not send directly to tempo-tempo-distributor:4317 in this gateway-enabled setup. Its generated NetworkPolicy allowed OTLP traffic only from the gateway. Use gateway port 8090 with TLS, bearer authentication, and tenant write permission. Do not disable the policies to bypass that path.

---

## 9. Configure Istio Tracing and Metrics Together

Back up the existing control-plane configuration:

```bash
oc get istio default -o yaml > work/istio-default.before.yaml
```

For a fresh baseline without extension providers, apply:

```bash
oc patch istio default --type merge -p '
spec:
  values:
    meshConfig:
      enableTracing: true
      extensionProviders:
        - name: tempo
          opentelemetry:
            service: otel-collector.bookinfo.svc.cluster.local
            port: 4317
'
```

If other extension providers exist, preserve them using `oc edit istio default`. This merge patch replaces the whole extensionProviders array; it does not merge provider entries by name.

Correct nesting: `spec.values.meshConfig.extensionProviders[].opentelemetry.service` and `port`. Neither belongs under `opentelemetry.grpc`.

Create a single default Telemetry resource with both metrics and tracing:

```bash
oc apply -f - <<'EOF'
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system
spec:
  metrics:
    - providers:
        - name: prometheus
  tracing:
    - providers:
        - name: tempo
      randomSamplingPercentage: 100
      enableIstioTags: true
EOF

oc get istio default
oc get telemetry -A
```

Checkpoint: Istio remains healthy. 100% tracing is for this exercise; tune sampling for production.

For the already configured session only: earlier instructions created a second selector-less enable-prometheus-metrics resource. Once the combined mesh-default above is applied, remove only that duplicate:

```bash
# Session-upgrade cleanup; unnecessary on a fresh installation.
oc delete telemetry enable-prometheus-metrics -n istio-system --ignore-not-found
```

Use one selector-less namespace default, and inspect any workload-specific policies before changing behavior.

Fresh Bookinfo workloads deployed in section 11 receive the current settings. If workloads already existed, let configuration reconcile and restart only specific application deployments if required; avoid restarting every deployment in the namespace, which also restarts the gateway/collector.

References: [Service Mesh tracing](https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.4/html/observability/distributed-tracing-and-service-mesh), [Istio Telemetry API](https://istio.io/latest/docs/reference/config/telemetry/).

---

## 10. Configure Prometheus Scraping

### 10.1 Istiod ServiceMonitor

```bash
oc apply -f - <<'EOF'
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: istiod-monitor
  namespace: istio-system
spec:
  targetLabels: [app]
  selector:
    matchLabels:
      istio: pilot
  endpoints:
    - port: http-monitoring
      interval: 30s
EOF
```

### 10.2 Proxy PodMonitors

OpenShift user-workload monitoring restricts a monitor to its own namespace. Repeat the monitor in every mesh workload namespace, including a separate gateway namespace if you use one.

This uses the annotation-based merged endpoint that worked during the session. The port-name filter is a consolidation improvement to avoid multiple targets for one proxy when it declares several ports.

```bash
for MONITOR_NS in bookinfo istio-system; do
  oc apply -n "$MONITOR_NS" -f - <<'EOF'
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: istio-proxies-monitor
spec:
  selector:
    matchExpressions:
      - key: istio-prometheus-ignore
        operator: DoesNotExist
  podMetricsEndpoints:
    - path: /stats/prometheus
      interval: 30s
      relabelings:
        - action: keep
          sourceLabels: [__meta_kubernetes_pod_container_name]
          regex: istio-proxy
        - action: keep
          sourceLabels: [__meta_kubernetes_pod_container_port_name]
          regex: http-envoy-prom
        - action: keep
          sourceLabels: [__meta_kubernetes_pod_annotationpresent_prometheus_io_scrape]
          regex: "true"
        - action: replace
          sourceLabels: [__meta_kubernetes_pod_annotation_prometheus_io_port, __meta_kubernetes_pod_ip]
          regex: '(\d+);(([A-Fa-f0-9]{1,4}::?){1,7}[A-Fa-f0-9]{1,4})'
          replacement: '[$2]:$1'
          targetLabel: __address__
        - action: replace
          sourceLabels: [__meta_kubernetes_pod_annotation_prometheus_io_port, __meta_kubernetes_pod_ip]
          regex: '(\d+);((([0-9]+?)(\.|$)){4})'
          replacement: '$2:$1'
          targetLabel: __address__
        - action: replace
          sourceLabels: [__meta_kubernetes_namespace]
          targetLabel: namespace
        - action: replace
          sourceLabels: [__meta_kubernetes_pod_label_app_kubernetes_io_name, __meta_kubernetes_pod_label_app]
          separator: ';'
          regex: '(.+);.*|.*;(.+)'
          replacement: '$1$2'
          targetLabel: app
        - action: replace
          sourceLabels: [__meta_kubernetes_pod_label_app_kubernetes_io_version, __meta_kubernetes_pod_label_version]
          separator: ';'
          regex: '(.+);.*|.*;(.+)'
          replacement: '$1$2'
          targetLabel: version
EOF
done

oc get servicemonitor -n istio-system
oc get podmonitor -n bookinfo
oc get podmonitor -n istio-system
```

Quoted heredocs are intentional: keep the delimiter quoted so shell expansion does not alter the Prometheus dollar replacements. Do not add extra escaping from unquoted-heredoc examples.

After Bookinfo is deployed, verify that the injected proxies have the expected port and annotations:

```bash
oc get pods -n bookinfo -o json | jq '.items[] | {
  name: .metadata.name,
  annotations: (.metadata.annotations | with_entries(select(.key | startswith("prometheus.io/")))),
  proxyPorts: [(.spec.containers[]?, .spec.initContainers[]?) |
    select(.name == "istio-proxy") | .ports[]?]
}'
```

Expected port name: `http-envoy-prom`; scrape annotation true; merged metrics port normally 15020. If your injection template differs, adapt the monitor to its real endpoint rather than adding duplicate monitors.

Reference: [Service Mesh metrics integration](https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.4/html/observability/metrics-and-service-mesh).

---

## 11. Deploy Bookinfo and Direct Ingress

### 11.1 Application

```bash
oc apply -n bookinfo \
  -f https://raw.githubusercontent.com/istio/istio/1.30.5/samples/bookinfo/platform/kube/bookinfo.yaml

for APP_DEPLOYMENT in details-v1 productpage-v1 ratings-v1 reviews-v1 reviews-v2 reviews-v3; do
  oc rollout status deployment/"$APP_DEPLOYMENT" -n bookinfo --timeout=180s
done

oc get pods -n bookinfo
```

Checkpoint: six application pods are 2/2 Running. The Collector is another deployment in the same namespace. If a future release changes sample locations, obtain the matching documented sample, not an arbitrary latest version.

### 11.2 GatewayClass

```bash
oc get gatewayclass istio -o yaml
```

Expected: accepted class using controller `istio.io/gateway-controller`. Istio normally supplies this class. If absent, inspect Istiod's Gateway API reconciliation before continuing.

The Sail values.ingressGateways field used in an earlier attempt was not available. You do not need istioctl to deploy this Gateway API gateway.

### 11.3 Gateway and HTTPRoute

```bash
oc apply -n bookinfo -f - <<'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: bookinfo-gateway
  labels:
    istio.io/rev: default
spec:
  gatewayClassName: istio
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: Same
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: bookinfo
spec:
  parentRefs:
    - name: bookinfo-gateway
      sectionName: http
  rules:
    - matches:
        - path:
            type: Exact
            value: /productpage
        - path:
            type: PathPrefix
            value: /static
        - path:
            type: Exact
            value: /login
        - path:
            type: Exact
            value: /logout
        - path:
            type: PathPrefix
            value: /api/v1/products
      backendRefs:
        - name: productpage
          port: 9080
EOF

oc wait --for=condition=Programmed \
  gateways.gateway.networking.k8s.io/bookinfo-gateway -n bookinfo --timeout=180s
oc get httproute bookinfo -n bookinfo -o yaml
oc get svc bookinfo-gateway-istio -n bookinfo
```

Check HTTPRoute parent conditions for `Accepted=True` and `ResolvedRefs=True`. Gateway `Programmed=True` alone does not prove AWS DNS, load-balancer backend health, or external connectivity.

### 11.4 External Endpoint and Browser

```bash
BOOKINFO_GATEWAY_HOST=$(oc get svc bookinfo-gateway-istio -n bookinfo \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

printf 'Bookinfo URL: http://%s/productpage\n' "$BOOKINFO_GATEWAY_HOST"
getent hosts "$BOOKINFO_GATEWAY_HOST"

curl -v --connect-timeout 10 --max-time 20 \
  "http://$BOOKINFO_GATEWAY_HOST/productpage" -o /dev/null
```

Checkpoint: non-empty hostname, successful DNS resolution, HTTP 200. AWS provisioning/DNS can take time. A private load balancer needs a client on a reachable network.

Open the printed URL using HTTP. Our gateway listener is port 80, with no TLS listener. A browser's automatic HTTPS upgrade can fail while HTTP curl succeeds.

For production, configure gateway HTTPS with your DNS name/certificate. Public HTTP is a demo choice, not the production transport configuration.

No manually created OpenShift Route is needed. An existing productpage Route from an earlier `oc expose` command is independent and bypasses the gateway unless explicitly retargeted. It is not generated by HTTPRoute.

### 11.5 Fresh Traffic

```bash
for i in {1..30}; do
  curl -sS --connect-timeout 10 --max-time 20 \
    -o /dev/null -w '%{http_code}\n' \
    "http://$BOOKINFO_GATEWAY_HOST/productpage"
done
```

Expected: HTTP 200. Do not suppress errors and infer success from no output.

For an internal gateway-only check:

```bash
oc exec -n bookinfo deploy/ratings-v1 -c ratings -- \
  curl -sS --max-time 10 -o /dev/null -w '%{http_code}\n' \
  http://bookinfo-gateway-istio.bookinfo.svc.cluster.local/productpage
```

That test verifies mesh routing, not the AWS ingress path.

---

## 12. Configure Kiali Once, with Both Data Sources

```bash
oc create clusterrolebinding kiali-monitoring-view \
  --clusterrole=cluster-monitoring-view \
  --serviceaccount=istio-system:kiali-service-account \
  --dry-run=client -o yaml | oc apply -f -

oc apply -f - <<'EOF'
apiVersion: kiali.io/v1alpha1
kind: Kiali
metadata:
  name: kiali
  namespace: istio-system
spec:
  auth:
    strategy: openshift
  deployment:
    cluster_wide_access: true
    logger:
      log_level: info
  external_services:
    grafana:
      enabled: false
    prometheus:
      url: https://thanos-querier.openshift-monitoring.svc.cluster.local:9091
      auth:
        type: bearer
        use_kiali_token: true
      thanos_proxy:
        enabled: true
    tracing:
      enabled: true
      provider: tempo
      use_grpc: false
      disable_version_check: true
      internal_url: https://tempo-tempo-gateway.tempo-system.svc.cluster.local:8080/api/traces/v1/dev/tempo
      health_check_url: https://tempo-tempo-gateway.tempo-system.svc.cluster.local:8080/api/traces/v1/dev/tempo/api/echo
      auth:
        type: bearer
        use_kiali_token: true
        insecure_skip_verify: false
EOF

oc rollout status deployment/kiali -n istio-system --timeout=180s
oc get deployment kiali -n istio-system \
  -o jsonpath='{.spec.template.spec.serviceAccountName}{"\n"}'
oc get route kiali -n istio-system \
  -o jsonpath='https://{.spec.host}{"\n"}'
```

Checkpoint: service account `kiali-service-account`, ready pod, HTTPS Kiali route.

**Corrections:**
- API group is `kiali.io`, not `kiali.org`.
- Use HTTP API mode with the gateway port 8080 and tenant-specific API path.
- `tracing.auth.ca_file` is deprecated and ignored by this Kiali release.
- Grafana is not deployed here, so its integration is disabled.

References: [Red Hat Kiali integration](https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.4/html/observability/kiali-operator-provided-by-red-hat), [Kiali CA bundles](https://v2-23.kiali.io/docs/configuration/p8s-jaeger-grafana/tls-configuration/), [Kiali reference](https://kiali.io/docs/configuration/kialis.kiali.io/).

---

## 13. Enable the Waterfall Viewer

After Cluster Observability Operator succeeds:

```bash
oc apply -f - <<'EOF'
apiVersion: observability.openshift.io/v1alpha1
kind: UIPlugin
metadata:
  name: distributed-tracing
spec:
  type: DistributedTracing
EOF

oc get uiplugin distributed-tracing -o yaml
```

Refresh the console when prompted. Open Observe → Traces.

| Field | Selection |
|---|---|
| Tempo instance | tempo in tempo-system |
| Tenant | dev |
| Time range | Last 15m initially |
| Service | productpage.bookinfo |

For the main page request, use Show query:

```traceql
{
  resource.service.name = "productpage.bookinfo"
  && span.http.url =~ ".*/productpage.*"
}
```

Reference: [Red Hat trace viewer](https://docs.redhat.com/en/documentation/red_hat_openshift_cluster_observability_operator/1-latest/html/ui_plugins_for_red_hat_openshift_cluster_observability_operator/distributed-tracing-ui-plugin).

---

## 14. Verify the Full Pipeline

### 14.1 Metrics Exist in Prometheus

```bash
oc exec -n openshift-user-workload-monitoring \
  prometheus-user-workload-0 -c prometheus -- \
  curl -sG http://localhost:9090/api/v1/query \
  --data-urlencode 'query=istio_requests_total{destination_service_namespace="bookinfo"}'
```

Expected: status success, non-empty result.

### 14.2 Collector Counters

```bash
oc exec -n bookinfo deploy/ratings-v1 -c ratings -- \
  curl -sS --max-time 10 \
  http://otel-collector-monitoring.bookinfo.svc.cluster.local:8888/metrics \
  | grep -E '^otelcol_(receiver_accepted_spans|exporter_sent_spans|exporter_send_failed_spans)'
```

Accepted and sent counts should be positive and grow with traffic.

### 14.3 Query Tempo Using Kiali's Identity

```bash
TEMPO_GATEWAY_URL=$(oc get route tempo-tempo-gateway -n tempo-system \
  -o jsonpath='https://{.spec.host}')

curl -sS -G --max-time 20 \
  -H "Authorization: Bearer $(oc create token kiali-service-account -n istio-system --duration=10m)" \
  "$TEMPO_GATEWAY_URL/api/traces/v1/dev/tempo/api/search" \
  --data-urlencode 'q={ resource.service.name = "productpage.bookinfo" }' \
  --data-urlencode 'limit=3' \
  -w '\nHTTP status: %{http_code}\n'
```

Expected: HTTP 200 and non-empty traces.

### 14.4 Check S3 Persistence Separately

```bash
oc get tempostack tempo -n tempo-system
oc get pvc -n tempo-system
oc logs -n tempo-system deployment/tempo-tempo-compactor --tail=60
oc logs -n tempo-system deployment/tempo-tempo-distributor --tail=60
aws s3 ls "s3://$TEMPO_BUCKET/" --recursive --summarize
```

Early searches can be served from ingester memory. S3 blocks appear after cutting/flushing.

---

## 15. Read Traces and Extend to Other Projects

### Reading the Verified Bookinfo Trace

The working browser trace had nine connected spans with HTTP 200:

```text
Gateway incoming operation                   74.31 ms
└─ Gateway outgoing request                   74.10 ms
   └─ Productpage incoming operation          72.93 ms
      ├─ Productpage → Details                 2.95 ms
      │  └─ Details incoming operation         1.55 ms
      └─ Productpage → Reviews                61.26 ms
         └─ Reviews v3 incoming operation     60.03 ms
            └─ Reviews → Ratings               3.42 ms
               └─ Ratings incoming operation   1.30 ms
```

### Multiple Projects and Centralized Collection

For each application namespace:

1. Apply the proper revision/discovery labels and deploy injected workloads.
2. Apply its proxy PodMonitor.
3. Allow access to the collector where policies restrict traffic.
4. Grant intended viewers namespace view access for query RBAC.
5. Verify correct per-span namespace metadata and application trace-context propagation.

For long-term centralized deployment, place the collector in observability and update the Istio provider FQDN, the collector writer binding's namespace, monitoring, and network policies accordingly.

---

## 16. Troubleshooting

| Symptom | Check or Correction |
|---|---|
| Kiali API resource mapping failure | Use `kiali.io/v1alpha1`. |
| Operator installed but no Kiali instance | Create the Kiali CR. |
| Prometheus unreachable | Thanos URL, bearer auth, monitoring RBAC, thanos_proxy.enabled. |
| HTTP 200 but empty Kiali graph | Query istio_requests_total; verify proxy scraping and fresh traffic within the time window. |
| Monitoring enabled but no metrics | User-workload monitoring alone does not create proxy scrape targets. |
| S3 mixed credential mode | Omit region from this static-key Secret. |
| S3 invalid endpoint | Include `https://`. |
| Gateway requires tenants / missing tenant mappings | Configure openshift mode plus tenantName and tenantId before starting gateway. |
| Collector timeouts at distributor | Use authenticated gateway:8090; do not bypass generated NetworkPolicies. |
| Token mount says account missing | Create an independent service account before the collector references it. |
| Collector label finds no pods | Use `instance=bookinfo.otel`. |
| Traces exist but Kiali has none | Inspect namespace enrichment, attribute visibility, exact query, and time range. |
| Only service.name and namespace visible | Query RBAC may be redacting attributes, not deleting storage data. |
| Kiali ca_file deprecation | Use supported CA-bundle trust; do not disable TLS validation. |
| No uiplugins API | Install COO, then its DistributedTracing UIPlugin. |
| Browser timeout while HTTP curl works | Check automatic HTTPS upgrades; this demo exposes HTTP:80. |

### Diagnostic Commands

```bash
oc get istio default
oc get tempostack tempo -n tempo-system
oc get pods -n bookinfo
oc get pods -n tempo-system
oc get events -n bookinfo --sort-by='.lastTimestamp'
oc logs -n bookinfo deployment/otel-collector -c otc-container --since=5m --tail=100
oc logs -n istio-system deployment/kiali --since=5m --tail=100
```

For Kiali query debugging:

```bash
oc patch kiali kiali -n istio-system --type merge \
  -p '{"spec":{"deployment":{"logger":{"log_level":"debug"}}}}'

# Refresh Kiali's Traces tab, then:
oc logs -n istio-system deployment/kiali --since=2m \
  | grep -Ei 'tempo|tracing|traceql|Get.*Traces|api/search'

# Restore normal logging after diagnosis:
oc patch kiali kiali -n istio-system --type merge \
  -p '{"spec":{"deployment":{"logger":{"log_level":"info"}}}}'
```

---

## 17. Completion Checklist

- [ ] Istio is healthy and application sidecars are ready.
- [ ] Supporting Red Hat operators succeeded.
- [ ] User-workload monitoring is healthy.
- [ ] Private S3 bucket and four-field static Secret are correct.
- [ ] TempoStack is ready; PVCs bind; objects eventually appear in S3.
- [ ] Collector identity exists independently and has tenant write access.
- [ ] Collector exports through the TLS/authenticated gateway.
- [ ] Per-span namespace enrichment is active for newly generated traces.
- [ ] One mesh-default Telemetry resource contains metrics and tracing.
- [ ] Per-namespace proxy monitors produce Prometheus data.
- [ ] Gateway/HTTPRoute reconcile and external HTTP requests return 200.
- [ ] Kiali graph and workload traces contain fresh data.
- [ ] Kiali's own service account can query traces with visible attributes.
- [ ] Observe → Traces shows the main /productpage waterfall.
- [ ] Temporary debug logging/exporters are removed.
- [ ] Production TLS, IAM/STS, tenancy, sampling, retention, capacity, and HA decisions are recorded before wider deployment.

---

## References

- [Service Mesh 3.4 installation and Gateway API](https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.4/html/installing/ossm-installing-service-mesh)
- [Service Mesh component versions](https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.4/html/release_notes/ossm-release-notes-version-support-tables)
- [Service Mesh metrics](https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.4/html/observability/metrics-and-service-mesh)
- [Service Mesh tracing](https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.4/html/observability/distributed-tracing-and-service-mesh)
- [Service Mesh Kiali integration](https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.4/html/observability/kiali-operator-provided-by-red-hat)
- [Tempo installation and authenticated export](https://docs.redhat.com/en/documentation/red_hat_openshift_distributed_tracing_platform/3.10/html/installing_distributed_tracing/distr-tracing-tempo-installing)
- [Tempo query RBAC](https://docs.redhat.com/en/documentation/red_hat_openshift_distributed_tracing_platform/3.9/html/configuring_distributed_tracing/distr-tracing-tempo-configuring)
- [OpenShift waterfall viewer](https://docs.redhat.com/en/documentation/red_hat_openshift_cluster_observability_operator/1-latest/html/ui_plugins_for_red_hat_openshift_cluster_observability_operator/distributed-tracing-ui-plugin)
- [Kiali trace features](https://v2-27.kiali.io/docs/features/tracing/)
- [Kiali trace troubleshooting](https://v2-27.kiali.io/docs/faq/distributed-tracing/)
- [Kiali CA bundles](https://v2-23.kiali.io/docs/configuration/p8s-jaeger-grafana/tls-configuration/)
- [Collector internal metrics](https://opentelemetry.io/docs/collector/internal-telemetry/)
- [Collector debugging](https://opentelemetry.io/docs/collector/troubleshooting/)
- [Tempo S3 static credentials and STS](https://grafana.com/docs/tempo/latest/set-up-for-tracing/setup-tempo/deploy/kubernetes/operator/object-storage/)
- [AWS bucket creation](https://docs.aws.amazon.com/cli/latest/reference/s3api/create-bucket.html)
- [AWS S3 IAM policies](https://docs.aws.amazon.com/AmazonS3/latest/userguide/example-policies-s3.html)
