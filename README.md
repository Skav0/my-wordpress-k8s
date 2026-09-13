# WordPress + MySQL on EKS

This repository deploys a simple WordPress application and a MySQL database on Amazon EKS using Kubernetes manifests and Kustomize.

The configuration in the repository matches the current manifests in `manifests/` and includes:

- a MySQL deployment and internal service
- a WordPress deployment
- an external WordPress `LoadBalancer` service
- persistent storage for both MySQL and WordPress
- a Kubernetes `Secret` for the database password
- a custom `StorageClass` for EBS-backed gp3 volumes

---

## Architecture

The stack has two main layers:

- Frontend: `wordpress` deployment running the official WordPress container
- Database: `wordpress-mysql` deployment running MySQL 8.0

Internal communication is handled by the service named `wordpress-mysql` on port `3306`.

The WordPress app is exposed externally through the service named `wordpress` with `type: LoadBalancer` and an AWS NLB class.

---

## Repository structure

```text
my-wordpress-k8s/
├── README.md
├── manifests/
│   ├── kustomization.yaml
│   ├── mysql.yaml
│   ├── pvc.yaml
│   ├── secret.yaml
│   ├── storageclass.yaml
│   ├── wordpress.yaml
│   └── wpsvc.yaml
└── .gitignore
```

---

## Resources created by the manifests

### StorageClass

File: `manifests/storageclass.yaml`

Creates a default EBS-backed storage class:

- Name: `ebs-gp3-sc`
- Provisioner: `ebs.csi.eks.amazonaws.com`
- Type: `gp3`
- Default class: `true`
- Reclaim policy: `Delete`

### PersistentVolumeClaims

File: `manifests/pvc.yaml`

Creates:

- `mysql-pv-claim` - 10Gi for MySQL data
- `wp-pv-claim` - 10Gi for WordPress files

Both use the `ebs-gp3-sc` storage class.

### Secret

File: `manifests/secret.yaml`

Creates a secret named `mysql-pass` with:

- key: `password`
- value: `skavi-eks-testing-password!`

This secret is referenced by both MySQL and WordPress containers.

### MySQL

File: `manifests/mysql.yaml`

Creates:

- Service: `wordpress-mysql`
- Deployment: `wordpress-mysql`

The MySQL container uses:

- image: `mysql:8.0`
- database: `wordpress`
- username: `wordpress`
- root password from `mysql-pass`
- persistent storage at `/var/lib/mysql`

### WordPress

File: `manifests/wordpress.yaml`

This file defines the WordPress Service for external access:

- name: `wordpress`
- type: `LoadBalancer`
- AWS NLB class: `eks.amazonaws.com/nlb`
- port: `80`
- targetPort: `80`
- selector: `app=wordpress, tier=frontend`

It also includes the AWS annotations:

```yaml
service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
service.beta.kubernetes.io/aws-load-balancer-subnets: "subnet-0208dadb6a312e9bc, subnet-00a8dde6aa8716f31"
```

These hardcoded subnet IDs are used because tag-based subnet discovery was not working in the environment.

### WordPress deployment

File: `manifests/wpsvc.yaml`

Defines the WordPress deployment with:

- label selector `app=wordpress, tier=frontend`
- container image `wordpress:latest`
- environment variables:
  - `WORDPRESS_DB_HOST=wordpress-mysql:3306`
  - `WORDPRESS_DB_USER=wordpress`
  - `WORDPRESS_DB_NAME=wordpress`
  - `WORDPRESS_DB_PASSWORD` is loaded from the `mysql-pass` secret
- volume mount `/var/www/html`

---

## Deploy

From the repo root:

```bash
kubectl apply -k manifests/
```

This applies all resources included in `manifests/kustomization.yaml`.

---

## Verify

Check that the pods and services are up:

```bash
kubectl get pods,svc,pvc
```

Check the external WordPress service:

```bash
kubectl get svc wordpress -o wide
```

Check logs for WordPress:

```bash
kubectl logs -l app=wordpress -f
```

Check logs for MySQL:

```bash
kubectl logs -l app=wordpress,tier=mysql -f
```

---

## Notes

- The project is intended for an AWS EKS environment with EBS CSI and an AWS Load Balancer setup.
- The WordPress service uses a public-facing NLB and hardcoded subnet IDs for the VPC.
- This is a simple lab/demo setup and not a production-grade secret management pattern.
- For real production use, store database credentials in a secure secret manager such as AWS Secrets Manager, External Secrets Operator, or a cluster-managed secret workflow.

---

## Useful troubleshooting commands

```bash
kubectl describe svc wordpress
kubectl describe pod -l app=wordpress
kubectl describe pod -l app=wordpress,tier=mysql
kubectl get events --sort-by=.metadata.creationTimestamp
```

If the LoadBalancer does not provision correctly, verify IAM permissions, VPC subnets, and whether your AWS account is allowed to create ELBs in the target region.