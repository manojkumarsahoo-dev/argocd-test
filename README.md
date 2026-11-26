# Practice Lab: EKS Cluster + ArgoCD on AWS

Hands-on lab to create an Amazon EKS cluster with `eksctl`, install ArgoCD, expose it with a LoadBalancer, and log in. Repeat it as many times as you like.

**Environment used:** EC2 instance (Amazon Linux 2023), region `us-east-1`, node type `t3.medium`, 2 worker nodes.

---

## Prerequisites

- An EC2 instance (Amazon Linux 2023) with an **IAM role** (or configured AWS credentials) that can create EKS, EC2, VPC, IAM and CloudFormation resources
- AWS CLI available (`aws --version`)
- Internet access from the instance

---

## Step 1: Become root

```bash
sudo su -
```

## Step 2: Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl version --client
```

> The original lab used `storage.googleapis.com/kubernetes-release/...`. That is the old location, so `dl.k8s.io` above is the safer URL.

## Step 3: Install eksctl

```bash
curl --silent --location \
  "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
  | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

Expected: a version number (the lab used `0.232.0`).

## Step 4: Create the EKS cluster

```bash
eksctl create cluster --name test \
  --region us-east-1 \
  --node-type t3.medium
```

- Takes about **15 to 16 minutes** (two CloudFormation stacks: control plane, then node group).
- eksctl installs the add-ons `vpc-cni`, `kube-proxy`, `coredns`, `metrics-server`.
- The warnings about "Auto Mode" and "OIDC is disabled" are informational. You can ignore them for this lab.
- Success looks like: `EKS cluster "test" in "us-east-1" region is ready`

## Step 5: Connect kubectl to the cluster

```bash
aws eks update-kubeconfig --region us-east-1 --name test
kubectl get nodes
```

You should see 2 nodes in `Ready` state.

## Step 6: Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Known error and fix

In the original run, plain `kubectl apply` gave this error:

```
The CustomResourceDefinition "applicationsets.argoproj.io" is invalid:
metadata.annotations: Too long: may not be more than 262144 bytes
```

Cause: client-side apply stores the whole manifest in an annotation, and the ApplicationSet CRD is too big for it.
Fix: use **`--server-side`** (already included in the command above).

If you hit the error anyway, re-run only the apply command with `--server-side --force-conflicts`.

## Step 7: Check the pods

```bash
kubectl get pods -n argocd
```

- Right after install some pods show `CreateContainerConfigError` or `Init:0/1`. This is normal while secrets and init containers finish.
- Wait 2 to 3 minutes and run it again. All 7 pods should be `1/1 Running`:

```
argocd-application-controller-0
argocd-applicationset-controller-...
argocd-dex-server-...
argocd-notifications-controller-...
argocd-redis-...
argocd-repo-server-...
argocd-server-...
```

If `argocd-applicationset-controller` was missing its CRD earlier, it will restart a few times and settle once the CRD exists.

## Step 8: Check services

```bash
kubectl get svc -n argocd
```

All services are `ClusterIP` at this point, so ArgoCD is not reachable from outside the cluster yet.

## Step 9: Expose ArgoCD with a LoadBalancer

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc argocd-server -n argocd -w
```

- Wait until `EXTERNAL-IP` shows an `...elb.amazonaws.com` hostname, then press `Ctrl + C` to stop watching.
- Open `https://<EXTERNAL-IP>` in your browser. Accept the self-signed certificate warning.

## Step 10: Get the admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

- **Username:** `admin`
- **Password:** the output of the command above (it is different for every new install)

> Tip: run this as its own command and press Enter. In the original session it was pasted together with another command, which produced a typo (`echoho`).

---

## Quick verification checklist

- [ ] `kubectl get nodes` shows 2 nodes `Ready`
- [ ] `kubectl get pods -n argocd` shows all pods `Running`
- [ ] `argocd-server` service has an ELB hostname as `EXTERNAL-IP`
- [ ] ArgoCD login page opens in the browser
- [ ] You can log in with `admin` and the secret password

---

## Cleanup (important, to avoid AWS charges)

The EKS control plane, 2 EC2 nodes and the LoadBalancer all cost money while they run.

```bash
# Delete the LoadBalancer first so the ELB is removed cleanly
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "ClusterIP"}}'

# Delete the cluster (takes about 10 minutes)
eksctl delete cluster --name test --region us-east-1
```

Then confirm in the AWS Console that the CloudFormation stacks `eksctl-test-cluster` and `eksctl-test-nodegroup-...` are gone.

---

## Troubleshooting

| Problem | What to check |
|---|---|
| `eksctl create cluster` fails | CloudFormation console, or run `eksctl utils describe-stacks --region=us-east-1 --cluster=test` |
| `kubectl` cannot connect | Re-run `aws eks update-kubeconfig --region us-east-1 --name test` |
| Pods stuck in `Init` or `CreateContainerConfigError` | Wait 2 to 3 minutes, then `kubectl describe pod <name> -n argocd` |
| `EXTERNAL-IP` stays `<pending>` | Wait a few minutes. If still pending, check subnets and IAM permissions for ELB creation |
| Browser cannot open the ELB URL | Use `https://`, accept the certificate warning, and wait 2 to 3 minutes for DNS to propagate |
| Lost the admin password | Run the Step 10 command again (works until you change the password and delete the secret) |

---

## Practice ideas

1. Redo the whole lab from scratch with a different cluster name and region.
2. Install the ArgoCD CLI and log in with `argocd login`.
3. Create your first ArgoCD Application that points to a Git repo with Kubernetes manifests.
4. Try `kubectl port-forward svc/argocd-server -n argocd 8080:443` instead of a LoadBalancer.
