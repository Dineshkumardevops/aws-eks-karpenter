# Prerequisites
 - An Instance with aws cli access with necessary tools installed on it (kubectl, eksctl, helm)
 - Ref:- https://docs.aws.amazon.com/eks/latest/best-practices/karpenter.html

## install kubectl
curl -LO "https://dl.k8s.io/release/v1.33.0/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client

## install eksctl
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" -o eksctl.tar.gz
tar -xzf eksctl.tar.gz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version

## install helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version


# AWS EKS + Karpenter POC

## Objective

Demonstrate Karpenter’s ability to automatically provision EC2 nodes in an EKS cluster based on workload requirements, using NodePools and NodeClass resources.

---

## Configure Environment Variables

First, we’ll define some important settings, such as the Karpenter version, Kubernetes version, and the region where the cluster will be deployed. Adjust these settings carefully based on your requirements:

```bash
export KARPENTER_NAMESPACE="kube-system"
export KARPENTER_VERSION="0.35.0"
export K8S_VERSION="1.29"
export AWS_PARTITION="aws"
export CLUSTER_NAME="karpenter-demo"
export AWS_DEFAULT_REGION="us-east-1"
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
export TEMPOUT="$(mktemp)"
```

---

## Stop Setting Kubernetes Requests and Limits

Learn How

---

## Deploy EKS Cluster

The following script will deploy an EKS cluster using the eksctl tool and set up some Karpenter-related resources, like the EventBridge rules for monitoring spot interruption events. The script will enable Fargate support, which is how we’ll run Karpenter with a serverless approach for simplicity. We need a worker node running for hosting our Karpenter pods, and leveraging Fargate simplifies this initial bootstrapping step. IAM roles are configured automatically to allow Karpenter the appropriate IAM permissions to manage our EC2 instances. No changes are required to the script below, but administrators are encouraged to review the script to understand what is being executed by eksctl and how the ClusterConfig resource works:

```bash
# Download the CloudFormation template to create a KarpenterNodeRole
curl -fsSL https://raw.githubusercontent.com/aws/karpenter-provider-aws/v"${KARPENTER_VERSION}"/website/content/en/preview/getting-started/getting-started-with-karpenter/cloudformation.yaml > "${TEMPOUT}"

# Create the KarpenterNodeRole
aws cloudformation deploy \
  --stack-name "${CLUSTER_NAME}" \
  --template-file "${TEMPOUT}" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides "ClusterName=${CLUSTER_NAME}"

# Create a new EKS cluster
eksctl create cluster -f - <<EOF
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: ${CLUSTER_NAME}
  region: ${AWS_DEFAULT_REGION}
  version: "${K8S_VERSION}"
  tags:
    karpenter.sh/discovery: ${CLUSTER_NAME}

iam:
  withOIDC: true
  serviceAccounts:
    - metadata:
        name: karpenter
        namespace: "${KARPENTER_NAMESPACE}"
      roleName: ${CLUSTER_NAME}-karpenter
      attachPolicyARNs:
        - arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:policy/KarpenterControllerPolicy-${CLUSTER_NAME}

iamIdentityMappings:
  - arn: "arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}"
    username: system:node:{{EC2PrivateDNSName}}
    groups:
      - system:bootstrappers
      - system:nodes

fargateProfiles:
  - name: karpenter
    selectors:
      - namespace: "${KARPENTER_NAMESPACE}"
EOF

# Retrieve the new cluster's details
export CLUSTER_ENDPOINT="$(aws eks describe-cluster --name "${CLUSTER_NAME}" --query "cluster.endpoint" --output text)"
export KARPENTER_IAM_ROLE_ARN="arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:role/${CLUSTER_NAME}-karpenter"

# Print the cluster details to the screen
echo "Cluster endpoint: ${CLUSTER_ENDPOINT}"
echo "Karpenter IAM role: ${KARPENTER_IAM_ROLE_ARN}"

# Create Spot service-linked role if not already present
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com || true

```

---

## Deploy Karpenter

We’re ready to deploy Karpenter to our new cluster; use the Helm command below to begin the installation. There are additional flags available to customize the installation configuration:

```bash
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter --version "${KARPENTER_VERSION}" --namespace "${KARPENTER_NAMESPACE}" --create-namespace \
	--set "settings.clusterName=${CLUSTER_NAME}" \
	--set "settings.interruptionQueue=${CLUSTER_NAME}" \
	--set controller.resources.requests.cpu=1 \
	--set controller.resources.requests.memory=1Gi \
	--set serviceAccount.create=false \
	--set serviceAccount.name=karpenter \
	--wait
```

You can see Karpenter deployed by running:

```bash
kubectl describe deployment karpenter --namespace "${KARPENTER_NAMESPACE}"
```

---

## NodePools and NodeClass Resources

We need to set up some NodePools and NodeClass resources to define our desired worker node configuration. Running the command below will create two NodePools and a NodeClass object, with each NodePool containing a team-based taint:

```bash
cat <<EOF | envsubst | kubectl apply -f -
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: team-1
spec:
  template:
    spec:
      taints:
        - key: team-1-nodes
          effect: NoSchedule
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
      nodeClassRef:
        name: default
---
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: team-2
spec:
  template:
    spec:
      taints:
        - key: team-2-nodes
          effect: NoSchedule
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
      nodeClassRef:
        name: default
---
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2 # Amazon Linux 2
  role: "KarpenterNodeRole-${CLUSTER_NAME}"
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}"
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}"
EOF

```
- kubectl get nodepools.karpenter.sh
- kubectl get ec2nodeclasses.karpenter.k8s.aws

---

## Deploy Workloads

No EC2 instances have been deployed to our cluster yet. We can test whether Karpenter provisions EC2 nodes by deploying new pods:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: team-1-nginx
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: "0.5"
        memory: 300Mi
  tolerations:
  - key: "team-1-nodes"
    operator: "Exists"
    effect: "NoSchedule"
---
apiVersion: v1
kind: Pod
metadata:
  name: team-2-nginx
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: "0.5"
        memory: 300Mi
  tolerations:
  - key: "team-2-nodes"
    operator: "Exists"
    effect: "NoSchedule"
EOF

```

* Karpenter will detect pods unschedulable on Fargate nodes and provision EC2 nodes to fit the workloads.

---

## Validation

```bash
kubectl logs deploy/karpenter --namespace "${KARPENTER_NAMESPACE}" -f
kubectl get nodes
kubectl get pods
```

* `team-1-nginx` should schedule to `team-1` node
* `team-2-nginx` should schedule to `team-2` node
* Nodes are **dynamically provisioned by Karpenter**

---

## Conclusion

* Karpenter successfully automates EC2 node provisioning based on pod requirements.
* Supports multiple NodePools with taints and tolerations.
* Fargate simplifies bootstrapping of Karpenter pods.
* Nodes are terminated automatically when workloads are removed.
