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



> 🔄 Restart the `prometheus-server` pod after this update:

```bash
kubectl delete pod -l app=prometheus,component=server -n observability
```

---

## ✅ 3. Deploy Sample App + Service + Ingress (Port 8080)

```
kubectl apply -n observability -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: http-echo
        image: hashicorp/http-echo
        args:
          - "-listen=:8080"
          - "-text=hello from 8080"
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: sample-app
  labels:
    app: sample-app
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  selector:
    app: sample-app
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sample-app-ingress
  annotations:
    example.io/should_be_probed: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: sample.127.0.0.1.nip.io
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: sample-app
              port:
                number: 80
EOF

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
