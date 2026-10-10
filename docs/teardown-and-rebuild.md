# Teardown and Rebuild Runbook

How to shut the whole AWS stack down so it stops costing money, and how to bring it back.

**What runs where**

| Layer | Managed by | Lives in |
|---|---|---|
| VPC, EKS, node group, ECR, IAM, Argo CD, monitoring | Terraform | `projects/Infrastructure/` |
| Boutique app (7 services + Postgres) | Argo CD (auto-sync, prune, self-heal) | `gitops/` |
| Container images | GitHub Actions (`Boutique CI Pipeline`, manual trigger) | ECR, tagged with the commit SHA |

**Running cost:** roughly $5/day (EKS control plane ~$0.10/hr + one `m7i-flex.large` node + disks), region `us-east-1`.

---

## Teardown

Order matters. Kubernetes creates some AWS resources (the Postgres EBS disk) that Terraform does not know about. If you destroy the cluster first, they are orphaned and keep billing.

### 1. Stop Argo CD from recreating things

Self-heal is on, so anything you delete gets recreated within seconds unless the Application goes first.

```bash
kubectl delete application boutique -n argocd
```

### 2. Delete the app namespace (removes the Postgres EBS disk)

```bash
kubectl delete namespace boutique
kubectl get pv            # should list nothing (reclaim policy is Delete)
```

Confirm the disk is gone in AWS:

```bash
aws ec2 describe-volumes --region us-east-1 \
  --filters Name=tag:kubernetes.io/created-for/pvc/namespace,Values=boutique \
  --query 'Volumes[].[VolumeId,State]' --output text      # expect empty
```

### 3. Destroy the infrastructure

Run in a terminal that stays open for the whole run (about 10–15 minutes). Closing it mid-run leaves a stale lock and half-deleted resources.

```bash
cd projects/Infrastructure
terraform destroy -auto-approve 2>&1 | tee destroy.log
```

Expect `Destroy complete! Resources: 32 destroyed.` ECR repos use `force_delete = true`, so images are deleted with them.

### 4. Verify nothing is left billing

```bash
aws eks list-clusters --region us-east-1
aws ec2 describe-instances --region us-east-1 \
  --filters Name=instance-state-name,Values=running,pending --query 'Reservations[].Instances[].InstanceId'
aws ec2 describe-volumes --region us-east-1 --filters Name=status,Values=available --query 'Volumes[].VolumeId'
aws ec2 describe-vpcs --region us-east-1 --filters Name=cidr,Values=10.1.0.0/16 --query 'Vpcs[].VpcId'
aws ecr describe-repositories --region us-east-1 --query 'repositories[].repositoryName'
aws elb describe-load-balancers --region us-east-1 --query 'LoadBalancerDescriptions[].LoadBalancerName'
aws elbv2 describe-load-balancers --region us-east-1 --query 'LoadBalancers[].LoadBalancerName'
```

All should come back empty (`[]`).

**Keep** `projects/Infrastructure/terraform.tfstate`. It is git-ignored and local only; without it Terraform cannot manage or cleanly recreate the stack.

---

## Rebuild

### 1. Create the infrastructure

```bash
aws sts get-caller-identity          # confirm account 032265100995
cd projects/Infrastructure
terraform init
terraform plan                       # expect: 32 to add, 0 to change, 0 to destroy
terraform apply -auto-approve 2>&1 | tee apply.log
```

Takes 15–20 minutes. Keep the terminal open.

If the run is interrupted:
- A leftover `.terraform.tfstate.lock.info` with no `terraform` process running is harmless; the local lock is released when the process exits.
- If `terraform plan` then says the EKS cluster `is tainted, so must be replaced` but `aws eks describe-cluster --name eks-cluster --region us-east-1` shows `ACTIVE`, run `terraform untaint module.eks.aws_eks_cluster.eks` instead of letting it rebuild.

### 2. Connect kubectl

```bash
aws eks update-kubeconfig --name eks-cluster --region us-east-1
kubectl get nodes                    # 1 node, Ready
kubectl get pods -n argocd           # all Running
kubectl get pods -n monitoring       # all Running
```

### 3. Rebuild and push the images

The ECR repos are new and empty. In GitHub: **Actions → Boutique CI Pipeline → Run workflow** on `main`.

The pipeline builds all 7 images tagged with the commit SHA, pushes them to ECR, then commits the new tag into `gitops/k8s/**` (`ci: update image tags to <sha>`).

Then pull that commit locally:

```bash
git pull
grep -rh "image:" gitops/k8s | sort -u     # every service on the new SHA, account 032265100995
```

### 4. Deploy the app with Argo CD

```bash
kubectl apply -f gitops/argo-cd.yml
kubectl get application boutique -n argocd -w     # wait for Synced / Healthy
```

### 5. Create the databases (once per fresh cluster)

The restore job is intentionally **not** in `kustomization.yml`, so Argo CD never reruns it. Run it once after Postgres is up:

```bash
kubectl rollout status statefulset/boutique-postgres -n boutique
kubectl apply -f gitops/k8s/database/restore-job.yml
kubectl wait -n boutique --for=condition=complete job/boutique-db-restore --timeout=180s
kubectl exec -n boutique boutique-postgres-0 -- psql -U postgres -Atc \
  "select datname from pg_database where not datistemplate"
# expect: postgres, auth_db, boutique_db, orders_db, products_db, users_db
```

`ERROR: role "postgres" already exists` in the job log is expected and harmless.

The backend services crash-loop until the databases exist. They recover on their own, or restart them:

```bash
kubectl rollout restart -n boutique deploy/auth deploy/order-service deploy/orders deploy/product-service deploy/user-service
kubectl get pods -n boutique           # all Running
```

### 6. Access

| What | Command | URL | Login |
|---|---|---|---|
| Argo CD | `kubectl port-forward -n argocd svc/argocd-server 8080:443` | https://localhost:8080 | `admin` / `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' \| base64 -d` |
| Grafana | `kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3007:80` | http://localhost:3007 | `admin` / `prom-operator` |
| Prometheus | `kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090` | http://localhost:9090 | none |
| Alertmanager | `kubectl port-forward -n monitoring svc/kube-prometheus-stack-alertmanager 9093:9093` | http://localhost:9093 | none |
| Boutique app | see below | http://localhost:3000 | — |

The frontend bundle calls `http://localhost:3001/api` (gateway) and `http://localhost:3003` (product service) from the browser, so all three must be forwarded on exactly these ports:

```bash
kubectl port-forward -n boutique svc/frontend 3000:3000 &
kubectl port-forward -n boutique svc/gateway 3001:3001 &
kubectl port-forward -n boutique svc/product-service 3003:3003 &
```

Stop the local Docker stack first (`docker-compose down` in `projects/boutique-microservices`); it uses the same ports.

---

## Gotchas

- **`kubectl` without `-n boutique`** targets the `default` namespace, where none of the app lives.
- **Manual changes don't stick.** With self-heal on, Argo CD reverts any `kubectl edit/delete` in `boutique`. Change `gitops/` and push instead.
- **Removing a manifest from `gitops/kustomization.yml` deletes it from the cluster** (prune is on).
- **Image tags must match ECR.** If pods show `InvalidImageName` or `ImagePullBackOff`, compare `grep -rh image: gitops/k8s` with `aws ecr describe-images --repository-name <svc> --region us-east-1`.
- **Terraform apply/destroy is blocked in Claude Code auto mode.** Run them yourself (in a terminal that stays open), or add a permission rule.
