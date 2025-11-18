# Helm Mentor Guide — Complete Beginner

**Audience:** Freshers who know little or nothing about Helm or Kubernetes.

**Goal:** Explain *what Helm is*, *how to install it*, *Helm chart structure*, and *step-by-step instructions* (with YAML examples) to deploy both a simple stateless application and a stateful application (Postgres) into a Kubernetes cluster using Helm.

---

## Table of contents

1. What is Helm?
2. Why use Helm?
3. Prerequisites
4. Install Helm (all platforms)
5. Helm basic commands (init-free Helm v3+)
6. Helm chart structure explained (folder tree)
7. Create a sample chart (`helm create`) — walkthrough
8. Key files and templates explained line-by-line
9. Example: Deploy a stateless web app (Nginx + simple config)
10. Example: Deploy a stateful app (Postgres StatefulSet + PVC) with values
11. Helm package, share, and repository basics
12. Common Helm patterns & best practices
13. Troubleshooting tips & debugging
14. Further reading and resources

---

## 1) What is Helm?

Helm is a package manager for Kubernetes. It helps you define, install, and upgrade complex Kubernetes applications using reusable packages called **charts**.

* A *Chart* is a collection of files that describe a related set of Kubernetes resources.
* A *Release* is a running instance of a chart (chart + configuration) deployed in a Kubernetes cluster.

Analogy: Helm is to Kubernetes what `apt` or `yum` is to Linux distributions.

---

## 2) Why use Helm?

* Reusability: Share charts across teams.
* Templating: Avoid repetitive YAML; charts accept `values.yaml` to parameterize.
* Versioning: Charts are versioned and can be rolled back.
* Dependencies: Charts can depend on other charts (e.g., microservices + database).
* Lifecycle commands: `install`, `upgrade`, `rollback`, `uninstall`.

---

## 3) Prerequisites

* A running Kubernetes cluster (minikube, kind, k3s, EKS, GKE, AKS, etc.).
* `kubectl` configured to talk to the cluster (`kubectl get nodes` works).
* Helm v3.x (v2 used Tiller; v3 is client-only and current standard).

---

## 4) Install Helm (v3) — quick commands

### macOS (Homebrew)

```bash
brew install helm
```

### Linux (script)

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### Windows (Chocolatey)

```powershell
choco install kubernetes-helm
```

### Verify

```bash
helm version
# Example output: version.BuildInfo{Version:"v3.x.x", ...}
```

---

## 5) Helm basic commands (v3)

* `helm repo add <name> <url>` — add a chart repository.
* `helm repo update` — refresh repo index.
* `helm search repo <term>` — search charts.
* `helm create <chart-name>` — scaffold a new chart.
* `helm install <release-name> <chart-dir-or-repo/chart> -f values.yaml` — install chart.
* `helm upgrade --install <release-name> <chart>` — upgrade or install.
* `helm uninstall <release-name>` — remove release.
* `helm list` — list releases.
* `helm history <release>` — view revision history.
* `helm rollback <release> <revision>` — rollback release.
* `helm lint <chart-dir>` — validate chart for common issues.
* `helm package <chart-dir>` — package a chart into a `.tgz`.

---

## 6) Helm chart structure (typical)

```
mychart/
├── Chart.yaml          # Chart metadata (name, version, appVersion, description)
├── values.yaml         # Default configuration values (overridable)
├── charts/             # Chart dependencies (subcharts)
├── templates/          # Kubernetes manifest templates (go-template)
│   ├── _helpers.tpl    # helper template functions
│   ├── deployment.yaml # Deployment manifest template
│   ├── service.yaml    # Service manifest template
│   ├── NOTES.txt       # printed after install
│   └── ...
└── README.md           # optional documentation
```

Each file in `templates/` is a Kubernetes manifest with Go templating. At `helm install` time, Helm renders these templates using values from `values.yaml` and CLI-provided values.

---

## 7) Create a sample chart and walkthrough

```bash
helm create myapp
cd myapp
tree -a
```

`helm create` scaffolds a chart with sample `deployment`, `service`, `ingress`, `hpa`, `serviceaccount`, and tests.

Run `helm lint .` to validate scaffolding.

---

## 8) Key files explained (line-by-line highlights)

### Chart.yaml

* `apiVersion`: Helm Chart API version (v2 or v1 — keep defaults)
* `name`: chart name
* `version`: chart package version
* `appVersion`: application version (e.g., image tag)

### values.yaml

Default, user-editable variables. Example keys:

```yaml
replicaCount: 2
image:
  repository: nginx
  tag: stable
service:
  type: ClusterIP
  port: 80
resources: {}
```

Users override these values with `-f custom-values.yaml` or `--set key=value`.

### templates/deployment.yaml (concept)

A simplified template uses placeholders and conditionals:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "myapp.name" . }}
  template:
    metadata:
      labels:
        app: {{ include "myapp.name" . }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
```

Note: `{{ ... }}` are Go template expressions; `.` is the root context with `.Values`, `.Chart`, `.Release`, etc.

### templates/_helpers.tpl

This file stores commonly used template functions (e.g., `fullname`, `name`). Use DRY practices here.

---

## 9) Example: Deploy a stateless web app (Nginx)

**Folder tree** (minimal chart named `nginx-app`)

```
nginx-app/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    └── NOTES.txt
```

**values.yaml** (example)

```yaml
replicaCount: 2
image:
  repository: nginx
  tag: "1.23"
service:
  type: ClusterIP
  port: 80
```

**templates/deployment.yaml** (minimal)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "nginx-app.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "nginx-app.name" . }}
  template:
    metadata:
      labels:
        app: {{ include "nginx-app.name" . }}
    spec:
      containers:
        - name: nginx
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
```

**templates/service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "nginx-app.fullname" . }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.port }}
      protocol: TCP
  selector:
    app: {{ include "nginx-app.name" . }}
```

**Install**

```bash
helm install webtest ./nginx-app -f values.yaml
# or override single value
helm install webtest ./nginx-app --set replicaCount=3
```

**Verify**

```bash
kubectl get pods
kubectl get svc
helm list
```

**Upgrade**

Change `values.yaml` then:

```bash
helm upgrade webtest ./nginx-app -f values.yaml
```

**Uninstall**

```bash
helm uninstall webtest
```

---

## 10) Example: Deploy a stateful app — Postgres using StatefulSet + PVC

This is a simplified example of a Helm chart that deploys a single Postgres instance using a StatefulSet and persistent volume claim.

**Folder tree**

```
postgres-chart/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── statefulset.yaml
    ├── service.yaml
    ├── pvc.yaml
    └── secret.yaml
```

**values.yaml** (simplified)

```yaml
replicaCount: 1
image:
  repository: postgres
  tag: "15"
postgresUser: myuser
postgresPassword: "mysecretpass"
postgresDatabase: mydb
persistence:
  enabled: true
  accessMode: ReadWriteOnce
  size: 8Gi
service:
  port: 5432
```

**templates/secret.yaml**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "postgres-chart.fullname" . }}-secret
type: Opaque
stringData:
  POSTGRES_USER: {{ .Values.postgresUser }}
  POSTGRES_PASSWORD: {{ .Values.postgresPassword }}
  POSTGRES_DB: {{ .Values.postgresDatabase }}
```

**templates/pvc.yaml** (optional helper for dynamically creating PVC)

```yaml
{{- if .Values.persistence.enabled }}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-{{ include "postgres-chart.fullname" . }}
spec:
  accessModes:
    - {{ .Values.persistence.accessMode }}
  resources:
    requests:
      storage: {{ .Values.persistence.size }}
{{- end }}
```

**templates/statefulset.yaml**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ include "postgres-chart.fullname" . }}
spec:
  serviceName: {{ include "postgres-chart.fullname" . }}-headless
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "postgres-chart.name" . }}
  template:
    metadata:
      labels:
        app: {{ include "postgres-chart.name" . }}
    spec:
      containers:
        - name: postgres
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          envFrom:
            - secretRef:
                name: {{ include "postgres-chart.fullname" . }}-secret
          ports:
            - containerPort: {{ .Values.service.port }}
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
          - {{ .Values.persistence.accessMode }}
        resources:
          requests:
            storage: {{ .Values.persistence.size }}
```

**templates/service.yaml** (headless service for StatefulSet)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "postgres-chart.fullname" . }}-headless
spec:
  clusterIP: None
  selector:
    app: {{ include "postgres-chart.name" . }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.port }}
```

**Install**

```bash
helm install pg ./postgres-chart -f values.yaml
```

**Notes:**

* StatefulSets manage stable network IDs and stable storage using `volumeClaimTemplates`.
* When `persistence.enabled = true`, the PVCs will be created and bound to a PV (depending on your cluster storage class).
* For production, secure passwords with external secret management (e.g., HashiCorp Vault or sealed-secrets) rather than plain `values.yaml`.

---

## 11) Helm package and share

Package a chart:

```bash
helm package ./mychart
# produces mychart-0.1.0.tgz
```

You can host charts in a chart repository (e.g., ChartMuseum, Amazon S3 + index, GitHub Pages) or share `.tgz` files.

To install from repo:

```bash
helm repo add myrepo https://example.com/charts
helm repo update
helm install app myrepo/mychart
```

---

## 12) Common Helm patterns & best practices

* Keep templates small & focused; use `_helpers.tpl` for repeated strings.
* Keep secrets out of `values.yaml` for public repos. Use sealed-secrets or external secret managers.
* Use `helm lint` and `helm template` before installing to check output.
* Use `--set` for quick overrides; use `-f` for larger change files.
* Add `NOTES.txt` to give users post-install instructions.
* Pin chart versions when deploying from a repo (e.g., `helm install --version 1.2.3`).

---

## 13) Troubleshooting & debugging

* See rendered manifests without applying (good for debugging):

```bash
helm template ./mychart -f values.yaml
```

* Check release status and history:

```bash
helm status <release-name>
helm history <release-name>
```

* Inspect Kubernetes objects:

```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

* Common issues:

  * ImagePullBackOff: wrong image or missing pull secret.
  * PVC stuck in `Pending`: no matching storage class or insufficient capacity.
  * Templating errors: run `helm lint` and `helm template` to debug.

---

## 14) Extra tips for beginners

* Start with `kind` or `minikube` for a local playground.
* Use `helm install --dry-run --debug` to test rendering and catch template errors.
* Version control your chart (keep `values.yaml` and important templates in git).
* Practice by converting an existing `kubectl apply -f` set of manifests into a chart.

---

## 15) Quick Cheat-sheet commands

```bash
# Create scaffold
helm create mychart

# Lint
helm lint mychart

# render locally
helm template mychart -f values.yaml

# install
helm install myrelease mychart -f values.yaml

# upgrade
helm upgrade myrelease mychart -f values.yaml

# uninstall
helm uninstall myrelease

# list
helm list
```

---

## 16) Closing notes

This guide gave you the conceptual model, installation steps, a stateless example, and a stateful example (Postgres) with templates and values. Use `helm template` and `helm lint` heavily while learning. Helm is powerful — start small and iterate.
