configmap.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: base
  LOG_LEVEL: info


deployment.yml


apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: web
          image: nginx:1.25
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: app-config



base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-svc
spec:
  selector:
    app: web-app
  ports:
    - port: 80
      targetPort: 80


kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - configmap.yaml
  - deployment.yaml
  - service.yaml



cat <<'EOF' > overlays/dev/migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: busybox:1.36
          command: ["sh", "-c", "echo Running DB migration; sleep 3"]


---
cat <<'EOF' > overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: app-dev
resources:
  - ../../base
  - migration-job.yaml
images:
  - name: nginx
    newTag: 1.25-alpine
patches:
  - target:
      kind: ConfigMap
      name: app-config
    patch: |-
      - op: replace
        path: /data/LOG_LEVEL
        value: debug
      - op: add
        path: /metadata/annotations
        value:
          argocd.argoproj.io/sync-wave: "0"
  - target:
      kind: Deployment
      name: web-app
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 1
      - op: add
        path: /metadata/annotations
        value:
          argocd.argoproj.io/sync-wave: "2"
  - target:
      kind: Service
      name: web-app-svc
    patch: |-
      - op: add
        path: /metadata/annotations
        value:
          argocd.argoproj.io/sync-wave: "2"


-------
cat <<'EOF' > overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: app-prod
resources:
  - ../../base
images:
  - name: nginx
    newTag: 1.25
patches:
  - target:
      kind: ConfigMap
      name: app-config
    patch: |-
      - op: replace
        path: /data/LOG_LEVEL
        value: warn
  - target:
      kind: Deployment
      name: web-app
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 3

============================================================================
git init
git add .
git commit -m "Add lab5 Kustomize base with dev and prod overlays"
git branch -M main
git remote add origin <YOUR_GIT_REPO_URL>
git push -u origin main

=============================================================================
# Where: Local Machine (inside lab5-argocd-kustomize)
kubectl kustomize overlays/dev > /tmp/dev-rendered.yaml
kubectl kustomize overlays/prod > /tmp/prod-rendered.yaml

cat /tmp/dev-rendered.yaml
cat /tmp/prod-rendered.yaml
diff -u /tmp/dev-rendered.yaml /tmp/prod-rendered.yaml




=============================================================================
# Where: Kubernetes Cluster (via kubectl)
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=180s

=============================================================================
# Where: Machine with kubectl & argocd CLI
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# Port-forward in a separate terminal or background if accessing locally:
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Log in with CLI:
argocd login localhost:8080 --username admin --password <PASTE_PASSWORD> --insecure

# Register your Git repository:
argocd repo add <YOUR_GIT_REPO_URL> --name lab5-repo
argocd repo list

=============================================================================
# Where: Machine with argocd CLI
argocd app create web-app-dev \
  --repo <YOUR_GIT_REPO_URL> \
  --path overlays/dev \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace app-dev \
  --create-namespace \
  --sync-policy automated \
  --auto-prune \
  --self-heal

argocd app create web-app-prod \
  --repo <YOUR_GIT_REPO_URL> \
  --path overlays/prod \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace app-prod \
  --create-namespace

# Trigger initial sync on prod manually:
argocd app sync web-app-prod

# Check status of both apps:
argocd app list
=============================================================================

# Where: Kubernetes Cluster (via kubectl)
kubectl get all,cm -n app-dev
kubectl get events -n app-dev --sort-by=.lastTimestamp
=============================================================================
# Where: Kubernetes Cluster (via kubectl)
kubectl scale deployment web-app -n app-dev --replicas=4
kubectl get deployment web-app -n app-dev


=============================================================================
# Where: Machine with argocd CLI (or kubectl)
argocd app get web-app-dev
sleep 15
kubectl get deployment web-app -n app-dev
argocd app list

=============================================================================


=============================================================================
Final Check
=============================================================================
# Where: Machine with argocd CLI
argocd app list

# Where: Kubernetes Cluster (via kubectl)
kubectl get pods -n app-dev -n app-prod
kubectl get svc -n app-dev -n app-prod