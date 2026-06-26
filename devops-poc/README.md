# DevOps POC – CI/CD Pipeline with Jenkins, Docker, Kubernetes & Prometheus

> Node.js application fully containerised and deployed via a Jenkins pipeline to a
> Kubernetes cluster, with Prometheus + Grafana monitoring.

---

## Project Structure

```
devops-poc/
├── app/
│   ├── app.js              # Express app with /metrics endpoint
│   ├── app.test.js         # Jest unit tests
│   ├── package.json
│   ├── Dockerfile          # Multi-stage build (deps → test → production)
│   └── .env.example
├── k8s/
│   ├── 00-namespace.yaml
│   ├── 01-configmap-secret.yaml
│   ├── 02-deployment.yaml  # Rolling-update, probes, HPA, PDB
│   ├── 03-service-hpa-pdb.yaml
│   ├── 04-ingress.yaml
│   ├── 05-prometheus.yaml  # RBAC + scrape config + alert rules
│   └── 06-grafana.yaml     # Pre-provisioned datasource
└── Jenkinsfile             # Declarative pipeline (10 stages)
```

---

## CI/CD Flow

```
Developer git push
      │
      ▼
  GitHub Repo
      │
      ▼ (webhook / poll)
  Jenkins Pipeline
      │
      ├── 1. Checkout
      ├── 2. Install Dependencies
      ├── 3. Code Quality (Lint + Audit – parallel)
      ├── 4. Unit Tests (Jest + coverage)
      ├── 5. SonarQube Analysis (main branch only)
      ├── 6. Build Docker Image (multi-stage)
      ├── 7. Trivy Security Scan
      ├── 8. Push to Docker Hub
      ├── 9. Deploy to Kubernetes (kubectl rollout)
      └── 10. Smoke Test (/health probe)
```

---

## Quick Start

### 1 · Run Locally

```bash
cd app
cp .env.example .env
npm install
npm test          # run unit tests
npm start         # http://localhost:3000
```

### 2 · Build & Run with Docker

```bash
cd app
docker build -t devops-poc-app .
docker run -p 3000:3000 --env-file .env devops-poc-app

# Verify
curl http://localhost:3000/health
curl http://localhost:3000/metrics
```

### 3 · Deploy to Kubernetes

```bash
# Apply all manifests (namespace, configmap, deployment, service, HPA, ingress, monitoring)
kubectl apply -f k8s/

# Watch rollout
kubectl rollout status deployment/devops-poc-app -n devops-poc

# Check pods
kubectl get pods -n devops-poc

# Port-forward app (if no Ingress)
kubectl port-forward svc/devops-poc-app-svc 8080:80 -n devops-poc
# → http://localhost:8080

# Port-forward Prometheus
kubectl port-forward svc/prometheus-svc 9090:9090 -n devops-poc
# → http://localhost:9090

# Port-forward Grafana
kubectl port-forward svc/grafana-svc 3001:3000 -n devops-poc
# → http://localhost:3001  (admin / admin123)
```

---

## Jenkins Setup

### Required Credentials

| Credential ID       | Type            | Purpose                         |
|---------------------|-----------------|---------------------------------|
| `DOCKER_HUB_CREDS`  | Username/Password | Push images to Docker Hub     |
| `KUBECONFIG_SECRET` | Secret File     | kubectl access to K8s cluster   |
| `SONAR_TOKEN`       | Secret Text     | SonarQube authentication        |

### Required Plugins

- Pipeline, Git, NodeJS, Docker Pipeline, HTML Publisher, JUnit,
  SonarQube Scanner, AnsiColor, Slack Notification (optional)

### Global Tool Configuration

- **NodeJS** → name: `NodeJS-18`, version: 18.x

---

## Application Endpoints

| Endpoint   | Description                        |
|------------|------------------------------------|
| `GET /`    | App info (version, environment)    |
| `GET /health` | Liveness check                  |
| `GET /ready`  | Readiness check                 |
| `GET /metrics`| Prometheus metrics scrape       |

---

## Prometheus Alert Rules

| Alert                | Condition                         | Severity |
|----------------------|-----------------------------------|----------|
| `AppDown`            | Pod unreachable > 1 min           | critical |
| `HighCPUUsage`       | CPU > 80 % for 5 min              | warning  |
| `HighMemoryUsage`    | Memory > 80 % for 5 min           | warning  |
| `HighHttpErrorRate`  | 5xx rate > 5 % for 2 min          | critical |
| `SlowResponseTime`   | p95 latency > 1 s for 5 min       | warning  |

---

## Production Checklist

- [ ] Replace `emptyDir` volumes with `PersistentVolumeClaims`
- [ ] Store secrets in AWS Secrets Manager / Vault (use External Secrets Operator)
- [ ] Enable TLS on the Ingress via cert-manager
- [ ] Configure Alertmanager → PagerDuty / Slack
- [ ] Enable SonarQube quality-gate blocking in Jenkinsfile
- [ ] Set Trivy `--exit-code 1` to block builds on critical CVEs
- [ ] Add DORA metrics dashboard in Grafana
