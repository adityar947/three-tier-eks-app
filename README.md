# 🌐 Three-Tier Application on Amazon EKS

This project deploys a full-stack **React + Node.js + MongoDB** application to **Amazon EKS (Elastic Kubernetes Service)** using:

- Docker
- Kubernetes (k8s)
- AWS Load Balancer Controller (ALB)
- GitHub Actions (CI/CD)
- Prometheus & Grafana (Monitoring)

---

## 📐 Architecture Overview

| Layer     | Tech Stack           |
|-----------|----------------------|
| Frontend  | React (Dockerized)   |
| Backend   | Node.js (Express API)|
| Database  | MongoDB              |
| Platform  | Amazon EKS (t2.medium nodes) |
| Ingress   | AWS ALB Controller   |
| Monitoring| Prometheus & Grafana |

### Namespaces

- `three-tier`: App components
- `kube-system`: Controllers
- `workshop`: (Optional/testing)

---

## 🚀 Deployment Guide

---

### 🧱 Part 1: Create the EKS Cluster

```bash
eksctl create cluster \
  --name three-tier-cluster \
  --region us-west-2 \
  --node-type t2.medium \
  --nodes-min 2 \
  --nodes-max 2 \
  --managed
```

```bash
aws eks update-kubeconfig --name three-tier-cluster --region us-west-2
```

---

### 🔐 Part 2: IAM & Load Balancer Controller

```bash
eksctl utils associate-iam-oidc-provider \
  --region us-west-2 \
  --cluster three-tier-cluster \
  --approve
```

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicyThreeTier \
  --policy-document file://iam_policy.json
```

```bash
eksctl create iamserviceaccount \
  --cluster=three-tier-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicyThreeTier \
  --approve \
  --override-existing-serviceaccounts \
  --region=us-west-2
```

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=three-tier-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-west-2
```

---

### 🐳 Part 3: Dockerize and Push Images to Amazon ECR

```bash
# Frontend
docker build -t three-tier-frontend ./Application-Code/frontend
docker tag three-tier-frontend:latest <ACCOUNT_ID>.dkr.ecr.us-west-2.amazonaws.com/three-tier-frontend
docker push <ACCOUNT_ID>.dkr.ecr.us-west-2.amazonaws.com/three-tier-frontend

# Backend
docker build -t three-tier-backend ./Application-Code/backend
docker tag three-tier-backend:latest <ACCOUNT_ID>.dkr.ecr.us-west-2.amazonaws.com/three-tier-backend
docker push <ACCOUNT_ID>.dkr.ecr.us-west-2.amazonaws.com/three-tier-backend
```

---

### 📦 Part 4: Kubernetes Deployment

```bash
# Create namespaces
kubectl create namespace three-tier
kubectl create namespace workshop

# MongoDB
cd Kubernetes-Manifests-file/Database
kubectl apply -f secrets.yaml -n three-tier
kubectl apply -f pv.yaml -n three-tier
kubectl apply -f pvc.yaml -n three-tier
kubectl apply -f deployment.yaml -n three-tier
kubectl apply -f service.yaml -n three-tier

# Backend
cd ../Backend
kubectl apply -f deployment.yaml -n three-tier
kubectl apply -f service.yaml -n three-tier

# Frontend
cd ../Frontend
kubectl apply -f deployment.yaml -n three-tier
kubectl apply -f service.yaml -n three-tier
```

---

### 🌐 Part 5: Ingress with AWS ALB

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mainlb
  namespace: three-tier
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/backend-protocol: HTTP
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
    kubernetes.io/ingress.class: alb
spec:
  rules:
    - http:
        paths:
          - path: /*
            pathType: ImplementationSpecific
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

```bash
kubectl apply -f ingress.yaml
```

---

### 🔄 Part 6: GitHub Actions CI/CD

Add this to `.github/workflows/deploy.yml` (see full Medium guide for code).

---

### 📊 Part 7: Monitoring with Prometheus & Grafana

```bash
kubectl create ns monitoring

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/prometheus -n monitoring

helm repo add grafana https://grafana.github.io/helm-charts
helm install grafana grafana/grafana -n monitoring --set adminPassword=admin123 --set service.type=LoadBalancer
```

---

### 🛡 Part 8: Security Best Practices

- Use AWS Secrets Manager with CSI
- Use HTTPS + ACM certs
- Enable AWS WAF
- Apply NetworkPolicies
- Restrict IAM roles

---

### 🧹 Cleanup

```bash
kubectl delete ns three-tier monitoring workshop
helm uninstall grafana -n monitoring
helm uninstall prometheus -n monitoring
eksctl delete cluster --name three-tier-cluster --region us-west-2
```

---

### 🙌 Credits

Inspired by [LondheShubham153/TWSThreeTierAppChallenge](https://github.com/LondheShubham153/TWSThreeTierAppChallenge)