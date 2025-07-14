# 📘 Blackbox Exporter Setup on EKS

## ✅ 1. Install Blackbox Exporter via Helm

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus-blackbox-exporter prometheus-community/prometheus-blackbox-exporter \
  -n observability --create-namespace
```

---

## ✅ 2. Add Scrape Jobs to Prometheus Config

If using Prometheus installed via Helm, edit the `values.yaml` or patch the config directly using:

```bash
kubectl edit configmap prometheus-server -n observability
```

And under `extraScrapeConfigs`, add:

```yaml
  - job_name: 'blackbox-external-targets'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://www.google.com
          - https://www.github.com
          - https://www.stackoverflow.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: prometheus-blackbox-exporter.observability.svc.cluster.local:9115

  - job_name: 'blackbox-kubernetes-services'
    metrics_path: /probe
    params:
      module: [http_2xx]
    kubernetes_sd_configs:
      - role: service
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: prometheus-blackbox-exporter.observability.svc.cluster.local:9115
      - source_labels: [__param_target]
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        target_label: kubernetes_service_name

  - job_name: 'blackbox-kubernetes-ingresses'
    metrics_path: /probe
    params:
      module: [http_2xx]
    kubernetes_sd_configs:
      - role: ingress
    relabel_configs:
      - source_labels:
          [__meta_kubernetes_ingress_scheme, __address__, __meta_kubernetes_ingress_path]
        regex: (.+);(.+);(.+)
        replacement: ${1}://${2}${3}
        target_label: __param_target
      - target_label: __address__
        replacement: prometheus-blackbox-exporter.observability.svc.cluster.local:9115
      - source_labels: [__param_target]
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_ingress_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_ingress_name]
        target_label: ingress_name

  - job_name: 'blackbox-kubernetes-pods'
    metrics_path: /probe
    params:
      module: [http_2xx]
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: prometheus-blackbox-exporter.observability.svc.cluster.local:9115
      - source_labels: [__param_target]
        replacement: ${1}/health
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: kubernetes_pod_name
```

> 🔄 Restart the `prometheus-server` pod after this update:

```bash
kubectl delete pod -l app=prometheus,component=server -n observability
```

---

## ✅ 3. Deploy a Demo Pod for `/health` Probe

```yaml
# health-demo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: health-demo
  namespace: observability
spec:
  replicas: 1
  selector:
    matchLabels:
      app: health-demo
  template:
    metadata:
      labels:
        app: health-demo
      annotations:
        blackbox.io/probe: "true"
    spec:
      containers:
        - name: web
          image: hashicorp/http-echo
          args:
            - "-listen=:8080"
            - "-text=ok"
            - "-path=/health"
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: health-demo
  namespace: observability
spec:
  selector:
    app: health-demo
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
```

Apply it:

```bash
kubectl apply -f health-demo.yaml
```

---

## ✅ 4. Add Grafana Dashboard

1. Import Grafana dashboard ID `7587` (Blackbox Exporter).
2. Set data source as your Prometheus.
3. You should now see:

   * Probe success/failure
   * Response time
   * Target status (UP/DOWN)

---

## ✅ 5. Troubleshooting

* Ensure `prometheus-blackbox-exporter` is running:

  ```bash
  kubectl get pods -n observability -l app.kubernetes.io/name=prometheus-blackbox-exporter
  ```

* Test probe manually:

  ```bash
  curl "http://prometheus-blackbox-exporter.observability.svc.cluster.local:9115/probe?target=https://www.google.com&module=http_2xx"
  ```

* View metrics in Prometheus:

  ```
  probe_success
  probe_duration_seconds
  ```

---
