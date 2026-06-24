# TP — Kubernetes & Helm : Azure Vote App

## Prérequis

- [Minikube](https://minikube.sigs.k8s.io/docs/start/) installé et démarré
- [kubectl](https://kubernetes.io/docs/tasks/tools/) installé
- [Helm](https://helm.sh/docs/intro/install/) installé
- [Azure CLI](https://learn.microsoft.com/fr-fr/cli/azure/install-azure-cli) installé

---

## 1. Démarrage du cluster Kubernetes

```bash
minikube start
kubectl get nodes
```

Le cluster local Minikube simule un environnement AKS complet. Une fois démarré, le nœud doit apparaître en statut `Ready` :

```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   1m    v1.35.1
```

---

## 2. Installation d'Azure CLI

```bash
winget install Microsoft.AzureCLI
az login
```

Azure CLI permet de gérer les ressources Azure et de se connecter à un cluster AKS. Dans le cadre de ce TP, il est utilisé pour l'authentification et la gestion des quotas.

---

## 3. Installation de Helm

```bash
winget install Helm.Helm
helm version
```

Helm est le gestionnaire de packages pour Kubernetes. Il permet de déployer des applications complexes via des **charts** (ensembles de fichiers de configuration).

---

## 4. Ajout des repositories Helm

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

---

## 5. Installation de l'Ingress Controller NGINX

L'Ingress Controller NGINX expose les services Kubernetes vers l'extérieur via un seul point d'entrée (LoadBalancer).

```bash
kubectl create namespace ingress-nginx

helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --set controller.replicaCount=1
```

Vérification :

```bash
kubectl get pods --namespace ingress-nginx
```

---

## 6. Installation de Redis

Redis est utilisé comme base de données in-memory pour stocker les votes de l'application.

```bash
kubectl create namespace redis

helm install redis bitnami/redis --namespace redis --set auth.enabled=false --set replica.replicaCount=1 --set master.persistence.enabled=false
```

Vérification :

```bash
kubectl get pods --namespace redis
```

```
NAME               READY   STATUS    RESTARTS   AGE
redis-master-0     1/1     Running   0          48s
redis-replicas-0   1/1     Running   0          48s
```

---

## 7. Création du Helm Chart — Azure Vote App

### Structure du chart

```
azure-vote-chart/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    └── _helpers.tpl
```

### Chart.yaml

```yaml
apiVersion: v2
name: azure-vote-chart
description: Azure Vote App - TP AKS Helm
type: application
version: 0.1.0
appVersion: "1.0"
```

### values.yaml

```yaml
replicaCount: 1

image:
  repository: dockersamples/examplevotingapp_vote
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: "nginx"
  host: "vote.local"

redis:
  host: "redis-master.redis.svc.cluster.local"
  port: "6379"

vote:
  option1: "Cats"
  option2: "Dogs"
  title: "Azure Voting App"
```

### templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "azure-vote-chart.fullname" . }}
  labels:
    {{- include "azure-vote-chart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "azure-vote-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "azure-vote-chart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: azure-vote-front
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 80
          env:
            - name: REDIS
              value: "{{ .Values.redis.host }}"
            - name: REDIS_PORT
              value: "{{ .Values.redis.port }}"
            - name: VOTE1VALUE
              value: "{{ .Values.vote.option1 }}"
            - name: VOTE2VALUE
              value: "{{ .Values.vote.option2 }}"
            - name: TITLE
              value: "{{ .Values.vote.title }}"
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "250m"
              memory: "256Mi"
```

### templates/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "azure-vote-chart.fullname" . }}
  labels:
    {{- include "azure-vote-chart.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: 80
      protocol: TCP
  selector:
    {{- include "azure-vote-chart.selectorLabels" . | nindent 4 }}
```

### templates/ingress.yaml

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "azure-vote-chart.fullname" . }}
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: {{ .Values.ingress.className }}
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ include "azure-vote-chart.fullname" . }}
                port:
                  number: {{ .Values.service.port }}
{{- end }}
```

---

## 8. Déploiement du chart

```bash
kubectl create namespace azure-vote
helm install azure-vote ./azure-vote-chart --namespace azure-vote
```

Vérification :

```bash
kubectl get pods --namespace azure-vote
```

```
NAME                                           READY   STATUS    RESTARTS   AGE
azure-vote-azure-vote-chart-56fd69bc9f-657mc   1/1     Running   0          16s
```

### Accès à l'application

```bash
kubectl port-forward svc/azure-vote-azure-vote-chart 8080:80 --namespace azure-vote
```

Ouvrir **http://localhost:8080**

L'application Azure Vote App est accessible avec les options **Cats** et **Dogs** :

![Azure Vote App](screenshots/azure-vote.png)

---

## 9. BONUS — Installation de KubeCost

KubeCost permet de monitorer et optimiser les coûts d'un cluster Kubernetes.

```bash
helm repo add kubecost https://kubecost.github.io/cost-analyzer/
helm repo update
kubectl create namespace kubecost
helm install kubecost kubecost/cost-analyzer --namespace kubecost --set global.clusterId=minikube-tp --version 2.8.0
```

Vérification :

```bash
kubectl get pods --namespace kubecost
```

```
NAME                                          READY   STATUS    RESTARTS   AGE
kubecost-cost-analyzer-f9ddb8b4d-g8szm        4/4     Running   0          3m5s
kubecost-forecasting-6f486979f4-hcqbq         1/1     Running   0          3m5s
kubecost-grafana-79f588d857-98924             2/2     Running   0          3m5s
kubecost-prometheus-server-766f59d7df-grrmw   1/1     Running   0          3m5s
```

### Accès à KubeCost

```bash
kubectl port-forward --namespace kubecost deployment/kubecost-cost-analyzer 9090
```

Ouvrir **http://localhost:9090**

![KubeCost Dashboard](screenshots/kubecost.png)

---

## Récapitulatif des commandes Helm

```bash
# Lister toutes les releases
helm list -A

# Désinstaller une release
helm uninstall <release-name> -n <namespace>

# Mettre à jour une release
helm upgrade <release-name> ./<chart> -n <namespace>
```

---

## Architecture déployée

```
[Browser]
    │
    ▼
[Ingress NGINX] ──── namespace: ingress-nginx
    │
    ▼
[Azure Vote App] ─── namespace: azure-vote
    │
    ▼
[Redis Master] ───── namespace: redis
```
