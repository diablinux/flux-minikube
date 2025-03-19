# Crossplane for Runtime

To create an EC2 instance in your AWS account using Crossplane from a local Minikube cluster, follow these steps:

---

### **Prerequisites**
1. **Minikube Running**:

Clone the repo and follow the instructions:

   ```bash
gh repo clone diablinux/k8s-local-dev
```

1. **AWS Account Credentials**:
   - An AWS IAM user with **programmatic access** (access key ID and secret key).
   - Ensure the IAM user has permissions to create EC2 instances (e.g., `AmazonEC2FullAccess` policy).

---

### **Step 1: Install Crossplane on Minikube **
#### 1.1 Add Crossplane Helm Repo:
```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
```

#### 1.2 Install Crossplane:
```bash
helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system \
  --create-namespace \
  --wait
```

#### Verify Installation:
```bash
kubectl get pods -n crossplane-system
# Should show "crossplane" and "crossplane-rbac-manager" running.
```

---

### **Step 2: Install the AWS Provider**
#### 2.1 Install `provider-aws`:
```bash
kubectl apply -f - <<EOF
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-aws:v0.40.0
EOF
```

#### Wait for the provider to become healthy:
```bash
kubectl get providers
# Should show "INSTALLED" and "HEALTHY".
```

---

### **Step 3: Configure AWS Credentials**
#### 3.1 Create a Kubernetes Secret with AWS credentials:
Replace `<YOUR_AWS_ACCESS_KEY_ID>` and `<YOUR_AWS_SECRET_ACCESS_KEY>` with your IAM user’s credentials.

```bash
kubectl create secret generic aws-creds \
  -n crossplane-system \
  --from-literal=aws_access_key_id=<YOUR_AWS_ACCESS_KEY_ID> \
  --from-literal=aws_secret_access_key=<YOUR_AWS_SECRET_ACCESS_KEY>
```

#### 3.2 Create a `ProviderConfig` to link credentials to the AWS provider:
```bash
kubectl apply -f - <<EOF
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      name: aws-creds
      namespace: crossplane-system
      key: aws_access_key_id
    secretRef:
      name: aws-creds
      namespace: crossplane-system
      key: aws_secret_access_key
  region: us-east-1  # Replace with your desired AWS region
EOF
```

---

### **Step 4: Create an EC2 Instance**
#### 4.1 Define an `EC2Instance` resource:
Create a file `ec2-instance.yaml` with the following content (adjust values as needed):
```yaml
apiVersion: ec2.aws.upbound.io/v1beta1
kind: Instance
metadata:
  name: my-ec2-instance
spec:
  forProvider:
    region: us-east-1
    instanceType: t2.micro
    ami: ami-0c55b159cbfafe1f0  # Amazon Linux 2 AMI (us-east-1)
    keyName: my-keypair         # Replace with your EC2 key pair name
    subnetId: subnet-xxxxxxxx   # Replace with your subnet ID (optional)
    tags:
      Name: crossplane-demo
  providerConfigRef:
    name: default
```

#### Notes:
- `ami`: Use an AMI ID valid for your region (e.g., find Amazon Linux 2 AMIs [here](https://us-east-1.console.aws.amazon.com/ec2/v2/home?region=us-east-1#Images:visibility=public-images)).
- `keyName`: Must match an existing EC2 key pair in your AWS account.
- `subnetId`: Optional (Crossplane will use the default VPC/subnet if omitted).

#### 4.2 Apply the resource:
```bash
kubectl apply -f ec2-instance.yaml
```

---

### **Step 5: Verify the EC2 Instance**
#### 5.1 Check Crossplane resource status:
```bash
kubectl describe instance.ec2.aws.upbound.io/my-ec2-instance
# Look for "SYNCED" and "READY" statuses.
```

#### 5.2 Check AWS Console:
- Log in to the [AWS EC2 Console](https://console.aws.amazon.com/ec2) to see the running instance.

---

### **Step 6: Clean Up**
#### 6.1 Delete the EC2 instance:
```bash
kubectl delete -f ec2-instance.yaml
```

#### 6.2 Uninstall Crossplane:
```bash
helm delete crossplane -n crossplane-system
minikube delete
```

---

### **Troubleshooting**
- **Provider Issues**: Check logs of the AWS provider pod:
  ```bash
  kubectl logs -n crossplane-system $(kubectl get pods -n crossplane-system -l pkg.crossplane.io/provider=provider-aws -o name)
  ```
- **Permissions**: Ensure the IAM user has `AmazonEC2FullAccess` (or equivalent permissions).
- **Key Pair**: The `keyName` must exist in your AWS account.

---

This workflow demonstrates how Crossplane treats cloud resources (like EC2) as Kubernetes-native objects, enabling GitOps and SRE practices for infrastructure management.