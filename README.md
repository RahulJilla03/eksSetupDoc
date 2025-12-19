# eksSetupDoc
It contains eks setup
Hello World
# End-to-end EKS setup with installation, configuration, and deployment

You’ve already proven the flow, so here’s a clean, detailed, reproducible guide you can save. It covers installing tools, creating the cluster and nodes, verifying, deploying Nginx with YAML, exposing it, and common troubleshooting.

---

## Prerequisites and assumptions

- **OS:** Linux x86_64 (Amazon Linux 2023, RHEL/CentOS, Ubuntu)
- **Permissions:** IAM user/role with AdministratorAccess (or equivalent for EKS, EC2, IAM, VPC, CloudFormation)
- **Region:** us-east-1
- **Instance type availability:** Use t3.small for reliable capacity; add c7i/m7i later if capacity is available

---

## Install the tooling

### AWS CLI v2

```bash
# Install
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify
aws --version
```

- **What this does:** Installs official AWS CLI v2.  
- **Verify:** You should see something like `aws-cli/2.x.x`.

### eksctl (latest)

```bash
# Download latest release tarball
curl -s https://api.github.com/repos/eksctl-io/eksctl/releases/latest \
| grep "browser_download_url.*linux_amd64.tar.gz" \
| cut -d '"' -f 4 | wget -qi -

# Extract and install
tar -xzf eksctl_*_linux_amd64.tar.gz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Verify
eksctl version
```

- **What this does:** Installs eksctl, the official EKS lifecycle tool.  
- **Verify:** Prints eksctl version and Git commit.

### kubectl (EKS 1.29 client)

```bash
# Download kubectl matching EKS 1.29
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.29.0/2023-11-14/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin

# Verify
kubectl version --client
```

- **What this does:** Installs kubectl binary compatible with EKS 1.29.  
- **Verify:** Shows client version v1.29.x.

---

## Configure AWS credentials

```bash
aws configure
```

- **Access key / secret:** Enter your IAM credentials.
- **Default region:** `us-east-1`
- **Output:** `json`

Verification:

```bash
aws sts get-caller-identity
aws ec2 describe-availability-zones --region us-east-1 --output table
```

- **Why:** Confirms credentials and region access are valid.

---

## Create the EKS control plane (no nodes yet)

```bash
eksctl create cluster \
  --name my-eks \
  --region us-east-1 \
  --version 1.29 \
  --without-nodegroup
```

- **What happens:**
  - Creates a dedicated VPC, subnets, security groups.
  - Provisions EKS control plane and IAM roles.
  - Manages via CloudFormation stack `eksctl-my-eks-cluster`.

Verification:

```bash
aws cloudformation describe-stacks \
  --region us-east-1 \
  --query "Stacks[?StackName=='eksctl-my-eks-cluster'].Stacks[0].StackStatus" \
  --output text
# Or check in AWS Console: CloudFormation → Stacks → eksctl-my-eks-cluster (CREATE_COMPLETE)
```

- **Ready when:** Stack shows `CREATE_COMPLETE`.

---

## Add a managed node group (t3.small, fixed at 2 nodes)

Option A: CLI (quick)

```bash
eksctl create nodegroup \
  --cluster my-eks \
  --region us-east-1 \
  --name ng-t3 \
  --node-type t3.small \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 2
```

- **What happens:**
  - Creates an EKS Managed Node Group with an Auto Scaling Group.
  - Uses Amazon Linux 2023 AMI family by default for EKS 1.29.

Monitor:

```bash
aws cloudformation describe-stack-events \
  --stack-name eksctl-my-eks-nodegroup-ng-t3 \
  --region us-east-1 \
  --query "StackEvents[0:15].[ResourceStatus,ResourceType,LogicalResourceId,Timestamp]" \
  --output table
```

- **Ready when:** Nodegroup stack shows `CREATE_COMPLETE`.

Option B: YAML (reproducible)

```yaml
# cluster-t3.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-eks
  region: us-east-1
  version: "1.29"

availabilityZones: ["us-east-1a", "us-east-1b"]

managedNodeGroups:
  - name: ng-t3
    instanceType: t3.small
    desiredCapacity: 2
    minSize: 2
    maxSize: 2
    amiFamily: AmazonLinux2023
```

Apply:

```bash
eksctl create nodegroup -f cluster-t3.yaml
```

- **Why YAML:** Source of truth, version-controlled, reproducible across environments.

---

## Connect kubectl to the cluster

```bash
aws eks update-kubeconfig --name my-eks --region us-east-1
kubectl cluster-info
kubectl get nodes
```

- **Expected:** Two nodes in `Ready` state, Kubernetes version `v1.29.x`.

---

## Deploy Nginx via YAML and expose with LoadBalancer

### Deployment + Service YAML

```yaml
# nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

Apply:

```bash
kubectl apply -f nginx.yaml
```

Verify:

```bash
# Pods scheduled and running
kubectl get pods -o wide

# Service created and externalized
kubectl get svc nginx
```

- **Expected:**
  - Two pods `Running`, each on a different node.
  - Service `nginx` shows an `EXTERNAL-IP` (AWS NLB/CLB/ALB depending on config).  
  - Open the external IP in a browser to see the Nginx welcome page.

---

## Common verification and troubleshooting

- **Check NodeGroup status:**
  ```bash
  aws eks describe-nodegroup \
    --cluster-name my-eks \
    --nodegroup-name ng-t3 \
    --region us-east-1 \
    --query "nodegroup.{status:status,health:health}"
  ```
  - **Tip:** `issues: []` = healthy. If `FAILED`, read the message (capacity/IAM/subnet).

- **Check EC2 instances launched:**
  ```bash
  aws ec2 describe-instances \
    --filters "Name=tag:eksctl.cluster.name,Values=my-eks" \
    --region us-east-1 \
    --query "Reservations[].Instances[].{ID:InstanceId,State:State.Name,AZ:Placement.AvailabilityZone}" \
    --output table
  ```

- **Pods stuck in ContainerCreating:**
  - **Check events:** `kubectl describe pod <pod-name>`
  - **Check CNI:** `kubectl -n kube-system get pods`
  - **Check IAM permissions** for the node role (AmazonEKSWorkerNodePolicy, CNI policies usually handled by eksctl).

- **No EXTERNAL-IP on Service:**
  - **Wait 1–3 minutes**; ELB takes time.
  - Ensure public subnets for nodes or that your cluster has an AWS Load Balancer Controller if using ALB Ingress later.

- **Capacity issues with c7i/m7i:**
  - Use `t3.small` or `m5.large` temporarily.
  - Pin AZs with available capacity or broaden subnets.

---

## Optional: add a second node group (e.g., c7i.large) later

```yaml
# cluster-multi.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-eks
  region: us-east-1
  version: "1.29"

availabilityZones: ["us-east-1a", "us-east-1b"]

managedNodeGroups:
  - name: ng-t3
    instanceType: t3.small
    desiredCapacity: 2
    minSize: 2
    maxSize: 2
    amiFamily: AmazonLinux2023

  - name: ng-c7i
    instanceType: c7i.large
    desiredCapacity: 2
    minSize: 2
    maxSize: 2
    amiFamily: AmazonLinux2023
```

Apply the additional group:

```bash
eksctl create nodegroup -f cluster-multi.yaml --only-nodegroup ng-c7i
```

- **Tip:** If capacity is unavailable, this will hang or fail; keep `ng-t3` as your reliable baseline.

---

## Cleanup (optional)

```bash
# Delete Nginx app
kubectl delete -f nginx.yaml

# Delete node groups
eksctl delete nodegroup --cluster my-eks --region us-east-1 --name ng-t3
# (and any others, e.g., ng-c7i)

# Delete cluster
eksctl delete cluster --name my-eks --region us-east-1
```

- **Order matters:** Delete workloads → node groups → cluster to avoid orphaned resources.

---

## Quick checklist for notes

- **Install:** AWS CLI, eksctl, kubectl
- **Configure:** aws configure (region us-east-1)
- **Cluster:** eksctl create cluster --without-nodegroup
- **Nodes:** eksctl create nodegroup (t3.small, min=2, max=2)
- **Connect:** aws eks update-kubeconfig
- **Verify:** kubectl get nodes
- **Deploy:** kubectl apply -f nginx.yaml
- **Expose:** Service type LoadBalancer, check EXTERNAL-IP
- **Troubleshoot:** nodegroup health, EC2 capacity, pod events, CNI
- **Cleanup:** delete app → nodegroups → cluster

If you want, I can tailor these steps into a single bash script with idempotent checks and clear logs for interview scenarios.