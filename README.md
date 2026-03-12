# Kubernetes Debugging & Deployment Assignment

This repository contains the completed Kubernetes debugging assignment. All intentional bugs in the provided manifests and Helm chart have been identified, diagnosed, and fixed. The cluster has been verified working end-to-end with HTTPS on a local Kind cluster.

---

## ✅ What Was Completed

- ✅ Identified and fixed **5 bugs** (+ 1 bonus) across the manifests and Helm chart
- ✅ Created a local Kind cluster using Docker and kind
- ✅ Installed cert-manager via Helm
- ✅ Applied ClusterIssuer and Certificate for `nginx.local`
- ✅ Deployed the nginx application via the fixed Helm chart
- ✅ Installed the ingress-nginx controller
- ✅ Verified **HTTP** access via `curl http://localhost:8080`
- ✅ Verified **HTTPS** access via `curl -k https://nginx.local:4430`

---

## 📁 Files in This Repository

| File | Description |
|------|-------------|
| `cert-manager/cluster-issuer.yaml` | ClusterIssuer manifest — **fixed** deprecated API version |
| `cert-manager/certificate.yaml` | Certificate manifest for `nginx.local` |
| `charts/nginx-app/values.yaml` | Helm values — **fixed** port type and TLS secret name |
| `charts/nginx-app/Chart.yaml` | Helm chart metadata |
| `charts/nginx-app/templates/deployment.yaml` | Deployment template — **fixed** YAML indentation bug |
| `charts/nginx-app/templates/service.yaml` | Service template — **fixed** selector label mismatch |
| `charts/nginx-app/templates/ingress.yaml` | Ingress template — **fixed** deprecated API and backend syntax |
| `k8s-debug-report.docx` | Full written bug report (root causes, fixes, prevention) |
| `k8s-debugging-presentation.pptx` | 11-slide presentation summarising all bugs and fixes |
| `README.md` | This file |

---

## 🐛 Bugs Found & Fixed

| # | File | What Was Broken | Fix Applied |
|---|------|-----------------|-------------|
| 1 | `cert-manager/cluster-issuer.yaml` | `apiVersion: cert-manager.io/v1alpha2` — removed in cert-manager v1.0 | Changed to `cert-manager.io/v1` |
| 2 | `charts/nginx-app/values.yaml` | `port: "eighty"` — string instead of integer; `targetPort: 8080` — nginx listens on 80 not 8080 | Changed to `port: 80`, `targetPort: 80` |
| 3 | `charts/nginx-app/templates/deployment.yaml` | `ports:` block mis-indented outside the container spec | Fixed indentation so `ports` sits inside the container |
| 4 | `charts/nginx-app/templates/service.yaml` | Selector `app: nginx-app-prod` — `-prod` suffix matched no pods, giving zero Endpoints | Removed `-prod` to match Deployment labels |
| 5 | `charts/nginx-app/templates/ingress.yaml` | `networking.k8s.io/v1beta1` removed in K8s 1.22; old backend syntax; missing `pathType` | Upgraded to `v1` with correct backend structure and `pathType: Prefix` |
| ★ | `certificate.yaml` vs `values.yaml` | Secret named `nginx-tls` in Certificate but `nginx-app-tls` in values — Ingress couldn't find the cert | Aligned both to `nginx-tls` |

---

## 🔍 How the Bugs Were Found

Each bug was found by carefully reading every file and cross-referencing them against each other:

1. **API version checks** — `v1alpha2` and `v1beta1` are known removed APIs in modern cert-manager and Kubernetes
2. **Type checking** — port values must be integers, not strings
3. **Indentation analysis** — counting YAML spaces to verify `ports:` was inside the container spec
4. **Label tracing** — comparing the Service selector against the Deployment pod labels to spot the `-prod` mismatch
5. **Secret name tracing** — grepping `secretName` across all files to catch the `nginx-tls` vs `nginx-app-tls` mismatch

---

## 🔗 Verified Working Commands

HTTP access:
```bash
kubectl port-forward svc/nginx-app-service 8080:80 -n nginx &
curl http://localhost:8080
```

HTTPS access:
```bash
kubectl port-forward svc/ingress-nginx-controller 4430:443 -n ingress-nginx &
curl -k https://nginx.local:4430
```

Both returned the nginx welcome page confirming the full end-to-end flow works. ✅

---

## 📄 Deliverables

- **`k8s-debug-report.docx`** — Written report covering what was broken, how it was diagnosed, root causes, fixes applied, tools used, and prevention recommendations
- **`k8s-debugging-presentation.pptx`** — 11-slide presentation summarising all bugs, fixes, and prevention strategies
- **Fixed manifests & Helm chart** — All YAML files in this repository reflect the corrected, fully working state
