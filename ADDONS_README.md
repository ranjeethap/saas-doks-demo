# Add-ons Pack (30‑min Demo)

## Install controllers (once)
```bash
# Ingress-NGINX
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace

# cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm upgrade --install cert-manager jetstack/cert-manager -n cert-manager --create-namespace --set installCRDs=true

# Prometheus (minimal)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install kps prometheus-community/kube-prometheus-stack -n monitoring --create-namespace -f k8s/monitoring/prometheus-values-min.yaml

# Flagger + loadtester
helm repo add flagger https://flagger.app
helm repo update
helm upgrade --install flagger flagger/flagger -n flagger --create-namespace --set meshProvider=nginx --set metricsServer=http://kps-kube-prometheus-stack-prometheus.monitoring:9090
helm upgrade --install flagger-loadtester flagger/loadtester -n dev --create-namespace
```

## ClusterIssuer
```bash
kubectl apply -f k8s/issuer-cluster.yaml
```

## Prod Ingress (sslip.io)
```bash
ING_IP=$(kubectl -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export HOST=app.${ING_IP}.sslip.io
envsubst < k8s/prod/ingress.yaml | kubectl apply -f -
kubectl -n prod get ingress
echo https://$HOST
```

## SealedSecrets quick demo
```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/latest/download/controller.yaml
brew install kubeseal
kubectl -n prod create secret generic db-creds --from-literal=username=demo --from-literal=password=S3cret! -o yaml --dry-run=client > /tmp/db-secret.yaml
kubeseal --format yaml < /tmp/db-secret.yaml > k8s/sealedsecrets-db-creds.yaml
kubectl apply -f k8s/sealedsecrets-db-creds.yaml
```

## External Secrets (optional demo)
```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm upgrade --install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace
kubectl -n external-secrets create secret generic backing-store --from-literal=db_username=demo --from-literal=db_password=S3cret!
kubectl apply -f k8s/eso/clustersecretstore.yaml
kubectl apply -f k8s/eso/externalsecret-db-creds.yaml
```

## Flagger canary
```bash
kubectl apply -f k8s/dev/canary.yaml
kubectl -n dev describe canary/doks-flask
kubectl -n dev get events --sort-by=.lastTimestamp | tail -n 20
```
