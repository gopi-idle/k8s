# DevOps EKS, Helm, MySQL, Redis, ArgoCD Guide

This document is a structured guide based on a conversation that covers **AWS EKS setup, Helm deployments, MySQL, Redis, ArgoCD usage, and troubleshooting scenarios**.

---

## 1. **EKS Cluster Setup**

### 1.1 Creating EKS Cluster via AWS Console

* Choose **Custom Configuration** for more control.
* Enable/Disable **Node Auto Repair** depending on requirement.

### 1.2 Common Issues

* **kubectl command not found**: Install kubectl properly.
* **kubectl syntax error (`<?xml version`)**: Wrong downloaded kubectl file, re-download from official source.
* **AccessDeniedException for EKS commands**: Ensure proper IAM role with `AmazonEKSClusterPolicy` and `AmazonEKSWorkerNodePolicy`.
* **ResourceNotFoundException**: Ensure cluster name is correct and exists in the region.

### 1.3 Connecting to EKS

```bash
aws eks update-kubeconfig --name <cluster-name> --region <region>
kubectl get nodes
```

---

## 2. **Helm Installation & Usage**

### 2.1 Install Helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### 2.2 Deploy MySQL using Helm

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-mysql bitnami/mysql --namespace mysql
```

### 2.3 Troubleshooting MySQL Pod Pending

* Check PersistentVolumeClaim status:

```bash
kubectl get pvc -n mysql
kubectl describe pod my-mysql-0 -n mysql
```

* Pending usually due to **no matching storage class**.
* Solution: Create StorageClass with `volumeBindingMode: Immediate`.

### 2.4 Create StorageClass YAML

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp2-immediate
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

### 2.5 Remove PVC

```bash
kubectl delete pvc <pvc-name> -n <namespace>
```

---

## 3. **AWS EFS Integration**

### 3.1 Create EFS

```bash
aws efs create-file-system --creation-token my-efs --region us-east-1
```

### 3.2 Delete EFS

```bash
aws efs delete-file-system --file-system-id <fs-id>
```

### 3.3 Mount EFS in Kubernetes

* Create PVC pointing to EFS.
* Use in MySQL/Redis deployment.

---

## 4. **Redis Deployment via Helm**

```bash
helm install my-redis bitnami/redis \
  --namespace redis \
  --set global.storageClass=efs-sc \
  --set master.persistence.existingClaim=efs-claim \
  --set replica.persistence.existingClaim=efs-claim
```

* Error `cannot re-use a name` occurs if release name exists. Use `helm uninstall my-redis` first.

---

## 5. **ArgoCD Installation**

### 5.1 Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 5.2 Access ArgoCD

* Default method: **Port-forward**

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

* NodePort method:

```bash
kubectl get svc argocd-server -n argocd
# Use <NodeIP>:<NodePort> in browser
```

### 5.3 ArgoCD App Creation Errors

* `app is not allowed in project`: use `default` project or create new project and assign repo and cluster permissions.

---

## 6. **Sample Nginx App Deployment via ArgoCD (Default Image)**

### 6.1 GitHub Repo Structure

```
nginx-sample-app/
└── k8s/
    ├── configmap.yaml
    ├── deployment.yaml
    └── service.yaml
```

### 6.2 `configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-html
  namespace: default
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>ArgoCD Nginx Demo</title>
    </head>
    <body>
        <h1>Hello from Nginx deployed via ArgoCD using ConfigMap!</h1>
    </body>
    </html>
```

### 6.3 `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-sample
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-sample
  template:
    metadata:
      labels:
        app: nginx-sample
    spec:
      containers:
        - name: nginx
          image: nginx:stable-alpine
          ports:
            - containerPort: 80
          volumeMounts:
            - name: html-volume
              mountPath: /usr/share/nginx/html
      volumes:
        - name: html-volume
          configMap:
            name: nginx-html
            items:
              - key: index.html
                path: index.html
```

### 6.4 `service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-sample-service
  namespace: default
spec:
  selector:
    app: nginx-sample
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: NodePort
```

### 6.5 GitHub Push & Deploy via ArgoCD

```bash
git init
git add .
git commit -m "Add Nginx app for ArgoCD"
git branch -M main
git remote add origin https://github.com/<username>/nginx-sample-app.git
git push -u origin main
```

* In ArgoCD UI → Create Application → select `default` project → Path: `k8s` → Sync

### 6.6 Access NodePort

```
http://<NodeIP>:<NodePort>
```

* Should show HTML page from ConfigMap

---

## 7. **Common Commands Summary**

* `kubectl get pods -n <namespace>`
* `kubectl get pvc -n <namespace>`
* `kubectl get svc -n <namespace>`
* `helm list -n <namespace>`
* `helm uninstall <release-name> -n <namespace>`
* `kubectl describe pod <pod-name> -n <namespace>`
* `kubectl delete pvc <pvc-name> -n <namespace>`
* `kubectl port-forward svc/<svc-name> -n <namespace> <local-port>:<svc-port>`

---

This document contains **step-by-step procedures, error outputs, and solutions** discussed in the conversation for **EKS, Helm, MySQL, Redis, and ArgoCD workflows**.
