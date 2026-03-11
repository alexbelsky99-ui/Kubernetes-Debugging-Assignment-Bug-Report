# Kubernetes Debugging & Deployment Assignment

A hands-on Kubernetes debugging exercise using **Kind**, **cert-manager**, **Helm**, and **nginx**. The provided manifests and Helm chart contained intentional bugs — this repository documents all issues found, fixes applied, and includes the full working configuration.

---

## 📁 Repository Structure

```
├── cert-manager/
│   ├── cluster-issuer.yaml           # ClusterIssuer for self-signed TLS (fixed)
│   └── certificate.yaml              # Certificate manifest for nginx.local
├── charts/
│   └── nginx-app/
│       ├── Chart.yaml
│       ├── values.yaml               # Fixed: port type + secret name
│       └── templates/
│           ├── deployment.yaml       # Fixed: indentation bug
│           ├── service.yaml          # Fixed: selector label
│           └── ingress.yaml          # Fixed: deprecated API + backend syntax
├── k8s-debug-report.docx             # Full written bug report
├── k8s-debugging-presentation.pptx  # 11-slide presentation
└── README.md
```

---

## 🐛 Bugs Found & Fixed

| # | File | Issue | Fix |
|---|------|-------|-----|
| 1 | `cert-manager/cluster-issuer.yaml` | Deprecated `apiVersion: cert-manager.io/v1alpha2` — removed in cert-manager v1.0 | Changed to `cert-manager.io/v1` |
| 2 | `charts/nginx-app/values.yaml` | `port: "eighty"` — string instead of integer; `targetPort: 8080` — nginx doesn't listen on 8080 | Changed to `port: 80`, `targetPort: 80` |
| 3 | `charts/nginx-app/templates/deployment.yaml` | `ports:` block mis-indented as a sibling of the container list item, not inside it | Fixed indentation so `ports` is a field inside the container spec |
| 4 | `charts/nginx-app/templates/service.yaml` | Selector `app: {{ .Release.Name }}-prod` — the `-prod` suffix matches no pod labels | Removed `-prod` suffix to align with Deployment labels |
| 5 | `charts/nginx-app/templates/ingress.yaml` | `networking.k8s.io/v1beta1` removed in K8s 1.22; old `serviceName`/`servicePort` backend syntax; missing `pathType` | Upgraded to `networking.k8s.io/v1` with correct backend structure and `pathType: Prefix` |
| ★ | `cert-manager/certificate.yaml` vs `values.yaml` | TLS secret named `nginx-tls` in Certificate but referenced as `nginx-app-tls` in values | Aligned both to `nginx-tls` |

---

## 🚀 Setup & Deployment

### Prerequisites

- [Docker Desktop](https://docs.docker.com/get-docker/)
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [helm v3](https://helm.sh/docs/intro/install/)

Install all three on Mac with Homebrew:
```bash
brew install kind kubectl helm
```

---

### 1. Create the Kind Cluster

```bash
kind create cluster --name k8s-debug
```

Verify it's healthy:
```bash
kubectl cluster-info --context kind-k8s-debug
kubectl get nodes
```

Expected output:
```
NAME                      STATUS   ROLES           AGE   VERSION
k8s-debug-control-plane   Ready    control-plane   75s   v1.35.0
```

---

### 2. Install cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

Wait for all 3 pods to be Running:
```bash
kubectl get pods -n cert-manager --watch
```

Expected output:
```
NAME                                      READY   STATUS    RESTARTS   AGE
cert-manager-79ffcfb98-zk7kt              1/1     Running   0          2m
cert-manager-cainjector-76c56f569-jxgcj   1/1     Running   0          2m
cert-manager-webhook-5b768c5858-m4wl7     1/1     Running   0          2m
```

---

### 3. Apply ClusterIssuer & Certificate

Create the nginx namespace first:
```bash
kubectl create namespace nginx
```

Then apply the manifests:
```bash
kubectl apply -f cert-manager/cluster-issuer.yaml
kubectl apply -f cert-manager/certificate.yaml
```

Expected output:
```
clusterissuer.cert-manager.io/selfsigned-cluster-issuer created
certificate.cert-manager.io/nginx-cert created
```

---

### 4. Deploy the nginx Application

```bash
helm install nginx-app ./charts/nginx-app \
  --namespace nginx
```

Verify the pod is running:
```bash
kubectl get pods -n nginx
```

---

### 5. Install the Ingress Controller

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

Wait for the pod to be Running:
```bash
kubectl get pods -n ingress-nginx --watch
```

---

## 🔗 Accessing the Application

### HTTP via port-forward

```bash
kubectl port-forward svc/nginx-app-service 8080:80 -n nginx &
curl http://localhost:8080
```

Expected output:
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and working.</p>
...
</html>
```

### HTTPS via Ingress ✅ Verified Working

```bash
# Add nginx.local to your hosts file
echo "127.0.0.1 nginx.local" | sudo tee -a /etc/hosts

# Forward the ingress controller port
kubectl port-forward svc/ingress-nginx-controller 4430:443 -n ingress-nginx &

# Test HTTPS
curl -k https://nginx.local:4430
```

Expected output:
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and working.</p>
...
</html>
```

---

## 🛡️ Prevention Recommendations

| Practice | Tool / Approach |
|----------|----------------|
| Validate API versions against target K8s version | `kubeconform` or `kubeval` in CI |
| Enforce value types in Helm charts | `values.schema.json` |
| Catch rendering + validation errors early | `helm install --dry-run --debug` in CI |
| Verify selector/label consistency | `helm template \| grep -E 'app:\|selector'` |
| Pin cert-manager versions | Lock chart + CRD versions in Helm |

---

## 🔧 Key Diagnostic Commands Used

```bash
kubectl apply --dry-run=client -f <manifest>   # Validate without applying
helm template ./charts/nginx-app               # Render chart to YAML
helm lint ./charts/nginx-app                   # Catch chart issues
kubectl get endpoints -n nginx                 # Check service → pod wiring
kubectl describe pod -n nginx                  # Diagnose container issues
kubectl logs -n nginx <pod>                    # Container log output
```

---

## 📄 Deliverables

- **`k8s-debug-report.docx`** — Written report covering what was broken, how it was diagnosed, root causes, fixes, tools used, and prevention recommendations.
- **`k8s-debugging-presentation.pptx`** — 11-slide presentation summarising all bugs and fixes.
- **Fixed manifests & Helm chart** — All files in this repository reflect the corrected, working state.
