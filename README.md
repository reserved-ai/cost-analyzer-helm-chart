# Overview
[Archera](https://archera.ai/) uses APIs built on top of [cost-model](https://github.com/kubecost/cost-model) agent to fetch cluster data. This data is then combined and enriched with other data sources to create a comprehensive view of their current and historical spending and resource allocation.

## Pre-Requisites:
- Kubernetes cluster v1.18+
- [Helm 3](https://github.com/helm/helm)
- Ingress controller preferably [nginx ingress controller](https://github.com/kubernetes/ingress-nginx).
- Container registry access token from Archera Team.

## Installation:
For now we are relying on open core Cost Analyzer agent and its recommended installation via helm:

1. Add Cost Analyzer helm chart repo:

```
helm repo add archera https://helm.archera.ai
helm repo update
```

2. Create Cost Analyzer namespace:

```
kubectl create namespace cost-analyzer
```

3. Obtain an access token from Archera team and create a k8s secret to store registry credentials:

```
kubectl -n cost-analyzer create secret docker-registry regcred \
--docker-server=ghcr.io --docker-username=archera-bot \
--docker-password=<container-registry-access-token>  \
--docker-email=bot@archera.ai
```

4. Obtain credentials for securing nginx ingress endpoint with your `access_token`:

```
curl -X GET https://api.archera.ai/v2/org/<org-id>/kubernetes/ingress/credentials
```

5. Create a k8s secret for storing obtained credentials:

```
htpasswd -c auth <username>
kubectl create secret generic basic-auth --from-file=auth --namespace=cost-analyzer
```

6. Create a file named `values.yaml` with following configuration:

```
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
    kubernetes.io/tls-acme: "true"
    # Adding basic authentication
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
  paths: ["/"]
  hosts:
  - <your-hostname>
  tls:
  - secretName: cost-analyzer-tls
    hosts:
    - <your-hostname>
```

Take a look below for detailed configuration options.

7. Install cost-analyzer helm chart

```
helm install cost-analyzer archera/cost-analyzer --namespace cost-analyzer --values values.yaml
```

8. Check the status of pods if they are all up and running in the `cost-analyzer` namespace:

```
kubectl get pods --namespace=cost-analyzer
```

Test if your endpoint is reachable by making a GET request.

9. Once the cost-analyzer is installed on your cluster, now you can register it with Archera using create cluster

```
curl -X POST https://api.archera.ai/v2/org/<org-id>/kubernetes/cluster/create
```

Example Request Body:
```
{
    "name": "test-cluster-1",
    "endpoint": "https://cost-analyzer.<your-hostname>"
}
```

Once the cluster is registered, allow some time(a couple of hours) for data aggregation scripts to trigger to fetch and aggregate data from your cluster.


## Configuration:
<a name="config-options"></a>
The following table lists commonly used configuration parameters for the Kubecost Helm chart and their default values.

Parameter | Description | Default
--------- | ----------- | -------
`global.prometheus.enabled` | If false, use an existing Prometheus install. [More info](http://docs.kubecost.com/custom-prom). | `true`
`prometheus.kube-state-metrics.disabled` | If false, deploy [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) for Kubernetes metrics | `false`
`prometheus.kube-state-metrics.resources` | Set kube-state-metrics resource requests and limits. | `{}`
`prometheus.server.persistentVolume.enabled` | If true, Prometheus server will create a Persistent Volume Claim. | `true`
`prometheus.server.persistentVolume.size` | Prometheus server data Persistent Volume size. Default set to retain ~6000 samples per second for 15 days. | `32Gi`
`prometheus.server.persistentVolume.storageClass` | Define storage class for Prometheus persistent volume  | `-`
`prometheus.server.retention` | Determines when to remove old data. | `15d`
`prometheus.server.resources` | Prometheus server resource requests and limits. | `{}`
`prometheus.nodeExporter.resources` | Node exporter resource requests and limits. | `{}`
`prometheus.nodeExporter.enabled` `prometheus.serviceAccounts.nodeExporter.create` | If false, do not crate NodeExporter daemonset.  | `true`
`prometheus.alertmanager.persistentVolume.enabled` | If true, Alertmanager will create a Persistent Volume Claim. | `true`
`prometheus.pushgateway.persistentVolume.enabled` | If true, Prometheus Pushgateway will create a Persistent Volume Claim. | `true`
`persistentVolume.enabled` | If true, Kubecost will create a Persistent Volume Claim for product config data.  | `true`
`persistentVolume.size` | Define PVC size for cost-analyzer  | `0.2Gi`
`persistentVolume.dbSize` | Define PVC size for cost-analyzer's flat file database  | `32.0Gi`
`persistentVolume.storageClass` | Define storage class for cost-analyzer's persistent volume  | `-`
`ingress.enabled` | If true, Ingress will be created | `false`
`ingress.annotations` | Ingress annotations | `{}`
`ingress.paths` | Ingress paths | `["/"]`
`ingress.hosts` | Ingress hostnames | `[cost-analyzer.local]`
`ingress.tls` | Ingress TLS configuration (YAML) | `[]`
`networkPolicy.enabled` | If true, create a NetworkPolicy to deny egress  | `false`
`networkCosts.enabled` | If true, collect network allocation metrics [More info](http://docs.kubecost.com/network-allocation) | `false`
`networkCosts.podMonitor.enabled` | If true, a [PodMonitor](https://github.com/coreos/prometheus-operator/blob/master/Documentation/api.md#podmonitor) for the network-cost daemonset is created | `false`
`serviceMonitor.enabled` | Set this to `true` to create ServiceMonitor for Prometheus operator | `false`
`serviceMonitor.additionalLabels` | Additional labels that can be used so ServiceMonitor will be discovered by Prometheus | `{}`
`prometheusRule.enabled` | Set this to `true` to create PrometheusRule for Prometheus operator | `false`
`prometheusRule.additionalLabels` | Additional labels that can be used so PrometheusRule will be discovered by Prometheus | `{}`
`grafana.resources` | Grafana resource requests and limits. | `{}`
`grafana.sidecar.datasources.defaultDatasourceEnabled` | Set this to `false` to disable creation of Prometheus datasource in Grafana | `true`
`serviceAccount.create` | Set this to `false` if you want to create the service account `kubecost-cost-analyzer` on your own | `true`
`tolerations` | node taints to tolerate | `[]`
`affinity` | pod affinity | `{}`
`extraVolumes` | A list of volumes to be added to the pod | `[]`|
`extraVolumeMounts` | A list of volume mounts to be added to the pod | `[]`
