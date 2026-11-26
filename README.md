# Practice Lab: EKS + ArgoCD GitOps Deployment (swiggy-app)

Hands-on lab to:

1. Create an Amazon EKS cluster with `eksctl`
2. Install ArgoCD and expose it with a LoadBalancer
3. Deploy an app (`swiggy-app`, nginx) from a Git repo using an ArgoCD Application with auto-sync
4. Test GitOps behavior (auto-sync, self-heal, prune)
5. Clean everything up to avoid AWS charges

**Environment used:** EC2 instance (Amazon Linux 2023), region `us-east-1`, node type `t3.medium`, 2 worker nodes.
**Git repo used:** `https://github.com/manojkumarsahoo-dev/argocd-test.git` (branch `main`, manifests in the `dev` folder)

---

## Prerequisites

- An EC2 instance (Amazon Linux 2023) with an **IAM role** (or configured AWS credentials) that can create EKS, EC2, VPC, IAM and CloudFormation resources
- AWS CLI available (`aws --version`)
- A GitHub repo (public is easiest) to hold your Kubernetes manifests
- Internet access from the instance

---

## Part 1: Create the EKS cluster

### Step 1: Become root

```bash
sudo su -
```

### Step 2: Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl version --client
```

> The original lab used `storage.googleapis.com/kubernetes-release/...`. That is the old location, so `dl.k8s.io` is the safer URL.

### Step 3: Install eksctl

```bash
curl --silent --location \
  "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
  | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

Expected: a version number (the lab used `0.232.0`).

### Step 4: Create the cluster

```bash
eksctl create cluster --name test \
  --region us-east-1 \
  --node-type t3.medium
```

- Takes about **15 to 16 minutes** (two CloudFormation stacks: control plane, then node group).
- eksctl installs the add-ons `vpc-cni`, `kube-proxy`, `coredns`, `metrics-server`.
- The warnings about "Auto Mode" and "OIDC is disabled" are informational. You can ignore them for this lab.
- Success looks like: `EKS cluster "test" in "us-east-1" region is ready`

### Step 5: Connect kubectl to the cluster

```bash
aws eks update-kubeconfig --region us-east-1 --name test
kubectl get nodes
```

You should see 2 nodes in `Ready` state.

---

## Part 2: Install ArgoCD

### Step 6: Create the namespace and install

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**Known error and fix:** plain `kubectl apply` gave this error in the original run:

```
The CustomResourceDefinition "applicationsets.argoproj.io" is invalid:
metadata.annotations: Too long: may not be more than 262144 bytes
```

Cause: client-side apply stores the whole manifest in an annotation, and the ApplicationSet CRD is too big for it.
Fix: use **`--server-side`** (already included above).

### Step 7: Check the pods

```bash
kubectl get pods -n argocd
```

- Right after install, some pods show `CreateContainerConfigError` or `Init:0/1`. This is normal while secrets and init containers finish.
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

### Step 8: Expose ArgoCD with a LoadBalancer

```bash
kubectl get svc -n argocd          # all ClusterIP at this point
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc argocd-server -n argocd -w
```

- Wait until `EXTERNAL-IP` shows an `...elb.amazonaws.com` hostname, then press `Ctrl + C`.
- Open `https://<EXTERNAL-IP>` in your browser. Accept the self-signed certificate warning.

### Step 9: Get the admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

- **Username:** `admin`
- **Password:** the output of the command above (different for every new install)

> Tip: run this as its own command and press Enter. In the original session it was pasted together with another command, which produced a typo (`echoho`).

---

## Part 3: Prepare the Git repo

Create this structure in your GitHub repo (`argocd-test`):

```
argocd-test/
├── README.md
└── dev/
    ├── deployment.yaml
    └── service.yaml
```

### `dev/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: swiggy-app
  labels:
    app: swiggy-app
spec:
  replicas: 4
  selector:
    matchLabels:
      app: swiggy-app
  template:
    metadata:
      labels:
        app: swiggy-app
    spec:
      containers:
        - name: swiggy-app
          image: nginx
          ports:
            - containerPort: 80
```

### `dev/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: swiggy-app
spec:
  type: LoadBalancer      # use ClusterIP if you do not want to expose it
  selector:
    app: swiggy-app
  ports:
    - port: 80
      targetPort: 80
```

Commit and push to the `main` branch.

---

## Part 4: Create the ArgoCD Application

You can do this from the UI or from a YAML file. Both give the same result.

### Option A: Using the ArgoCD UI

1. Log in to ArgoCD at `https://<EXTERNAL-IP>`
2. Click **+ NEW APP** and fill in:

| Field | Value |
|---|---|
| Application Name | `test` |
| Project | `default` |
| Sync Policy | `Automatic` (tick **Prune Resources** and **Self Heal**) |
| Sync Options | **Auto-Create Namespace** (and optionally **Server-Side Apply**) |
| Repository URL | `https://github.com/manojkumarsahoo-dev/argocd-test.git` |
| Revision | `main` |
| Path | `dev` |
| Cluster URL | `https://kubernetes.default.svc` |
| Namespace | `test` |

3. Click **CREATE**

### Option B: Using a YAML file

Save as `application.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: test
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/manojkumarsahoo-dev/argocd-test.git
    targetRevision: main
    path: dev
  destination:
    server: https://kubernetes.default.svc
    namespace: test
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

Apply it:

```bash
kubectl apply -f application.yaml
```

---

## Part 5: Verify the deployment

```bash
kubectl get applications -n argocd
kubectl get all -n test
```

Expected result in the ArgoCD UI (and CLI):

- App status: **Synced** and **Healthy**
- Resource tree: `test` → `svc swiggy-app`, `deploy swiggy-app` → `rs swiggy-app-xxxx` → **4 pods**, all `Running 1/1`

If you made the Service type `LoadBalancer`, get its address and open it in the browser to see the nginx welcome page:

```bash
kubectl get svc swiggy-app -n test
```

---

## Part 6: Test GitOps behavior (practice)

1. **Auto-sync:** change `replicas: 4` to `2` in `dev/deployment.yaml`, commit and push. ArgoCD detects the change and scales down (default polling is about 3 minutes, or click **Refresh**).
2. **Self-heal:** run `kubectl scale deployment swiggy-app -n test --replicas=1`. ArgoCD reverts it back to the value in Git.
3. **Pod recovery:** run `kubectl delete pod <pod-name> -n test`. The ReplicaSet creates a new one.
4. **Prune:** delete `service.yaml` from Git and push. ArgoCD removes the Service from the cluster.
5. **Rollback:** in the UI open **HISTORY AND ROLLBACK** and roll back to an earlier revision. (Auto-sync must be disabled first for a manual rollback.)

---

## Quick verification checklist

- [ ] `kubectl get nodes` shows 2 nodes `Ready`
- [ ] `kubectl get pods -n argocd` shows all pods `Running`
- [ ] `argocd-server` service has an ELB hostname as `EXTERNAL-IP`
- [ ] ArgoCD login page opens and `admin` login works
- [ ] Application `test` shows **Synced** and **Healthy**
- [ ] 4 `swiggy-app` pods are `Running 1/1` in namespace `test`
- [ ] Changing the repo (push) triggers an automatic sync

---

## Part 7: Cleanup (important, to avoid AWS charges)

The EKS control plane, 2 EC2 nodes and the LoadBalancers all cost money while they run. Follow this order:

```bash
# 1. Delete the ArgoCD application (removes swiggy-app and its LoadBalancer)
kubectl delete application test -n argocd

# 2. Switch ArgoCD's service back so its ELB is removed
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "ClusterIP"}}'

# 3. Confirm no LoadBalancer services remain
kubectl get svc -A | grep LoadBalancer

# 4. Delete the cluster (about 10 minutes)
eksctl delete cluster --name test --region us-east-1
```

Then confirm in the AWS Console that the CloudFormation stacks `eksctl-test-cluster` and `eksctl-test-nodegroup-...` are gone, and that no load balancers remain under **EC2 > Load Balancers**.

> If you skip steps 1 to 3, leftover load balancers can block VPC deletion and the cluster delete may fail.

---

## Troubleshooting

| Problem | What to check |
|---|---|
| `eksctl create cluster` fails | CloudFormation console, or `eksctl utils describe-stacks --region=us-east-1 --cluster=test` |
| `kubectl` cannot connect | Re-run `aws eks update-kubeconfig --region us-east-1 --name test` |
| CRD `Too long` error on ArgoCD install | Use `kubectl apply --server-side --force-conflicts` |
| ArgoCD pods stuck in `Init` or `CreateContainerConfigError` | Wait 2 to 3 minutes, then `kubectl describe pod <name> -n argocd` |
| `EXTERNAL-IP` stays `<pending>` | Wait a few minutes. If still pending, check subnets and IAM permissions for ELB creation |
| Browser cannot open the ELB URL | Use `https://`, accept the certificate warning, wait 2 to 3 minutes for DNS |
| App shows `OutOfSync` | Click **Refresh**, then **Sync**. Check the **DIFF** button to see what differs |
| App shows `Unknown` or repo error | Check the repo URL, branch and path (`dev`). Private repos need credentials added in ArgoCD Settings |
| Pods stay `Pending` | `kubectl describe pod <name> -n test` (often not enough node capacity) |
| Lost the admin password | Run the Step 9 command again (works until the secret is deleted) |

---

## Useful commands

```bash
kubectl get applications -n argocd          # list ArgoCD apps
kubectl describe application test -n argocd # sync details and errors
kubectl get all -n test                     # everything in the app namespace
kubectl logs -n argocd deploy/argocd-repo-server   # repo / manifest problems
eksctl get cluster --region us-east-1       # list clusters
```

---

## More practice ideas

1. Redo the whole lab from scratch with a different cluster name and region.
2. Install the ArgoCD CLI and log in with `argocd login`.
3. Use `kubectl port-forward svc/argocd-server -n argocd 8080:443` instead of a LoadBalancer.
4. Add a second app path (`prod`) and a second ArgoCD Application.
5. Replace the plain manifests with a Helm chart or Kustomize overlay.
