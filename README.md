# Comprehensive Guide: Multi-Environment RBAC Setup for AWS EKS and Self-Managed Kubernetes

## Part 1: AWS EKS Setup

### Step 1: Set up AWS CLI and eksctl
Ensure you have AWS CLI and eksctl installed and configured with appropriate permissions.

```bash
aws configure
eksctl version
```

### Step 2: Create an EKS cluster (if not already existing)
```bash
eksctl create cluster --name my-cluster --region us-west-2 --nodegroup-name standard-workers --node-type t3.medium --nodes 3 --nodes-min 1 --nodes-max 4
```

### Step 3: Create namespaces
```bash
kubectl create namespace dev
kubectl create namespace test
kubectl create namespace pre-prod
kubectl create namespace prod
```

### Step 4: Create IAM users for each environment
```bash
for env in dev test pre-prod prod; do
  aws iam create-user --user-name ${env}-user
done
```

### Step 5: Create IAM policies for each environment
Create policy documents for each environment. Here's an example for the dev environment (`dev-policy.json`):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "eks:DescribeCluster",
                "eks:ListClusters"
            ],
            "Resource": "*"
        }
    ]
}
```

Create similar policies for test, pre-prod, and prod, adjusting permissions as needed.

Apply the policies:
```bash
for env in dev test pre-prod prod; do
  aws iam create-policy --policy-name ${env}-EKSAccessPolicy --policy-document file://${env}-policy.json
done
```

### Step 6: Attach policies to users
```bash
for env in dev test pre-prod prod; do
  aws iam attach-user-policy --user-name ${env}-user --policy-arn arn:aws:iam::ACCOUNT_ID:policy/${env}-EKSAccessPolicy
done
```
Replace ACCOUNT_ID with your AWS account ID.

### Step 7: Create access keys for users
```bash
for env in dev test pre-prod prod; do
  aws iam create-access-key --user-name ${env}-user > ${env}-access-key.json
done
```
Save these access keys securely.

### Step 8: Configure aws-auth ConfigMap
Get the current aws-auth ConfigMap:
```bash
kubectl get configmap aws-auth -n kube-system -o yaml > aws-auth-configmap.yaml
```

Edit aws-auth-configmap.yaml and add the following under the `mapUsers` section:
```yaml
mapUsers: |
  - userarn: arn:aws:iam::ACCOUNT_ID:user/dev-user
    username: dev-user
    groups:
      - dev-group
  - userarn: arn:aws:iam::ACCOUNT_ID:user/test-user
    username: test-user
    groups:
      - test-group
  - userarn: arn:aws:iam::ACCOUNT_ID:user/pre-prod-user
    username: pre-prod-user
    groups:
      - pre-prod-group
  - userarn: arn:aws:iam::ACCOUNT_ID:user/prod-user
    username: prod-user
    groups:
      - prod-group
```
Replace ACCOUNT_ID with your AWS account ID.

Apply the updated ConfigMap:
```bash
kubectl apply -f aws-auth-configmap.yaml
```

### Step 9: Create Kubernetes RBAC resources
Create a file named `eks-rbac.yaml`:

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: dev-role
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-role-binding
  namespace: dev
subjects:
- kind: Group
  name: dev-group
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: dev-role
  apiGroup: rbac.authorization.k8s.io
---
# Repeat similar Role and RoleBinding for test, pre-prod, and prod namespaces
# Adjust permissions as needed for each environment
```

Apply the RBAC configuration:
```bash
kubectl apply -f eks-rbac.yaml
```

### Step 10: Configure kubectl for each user
For each environment, create a new kubectl context:

```bash
for env in dev test pre-prod prod; do
  aws eks get-token --cluster-name my-cluster --region us-west-2 --user-name ${env}-user > ${env}-token.txt
  kubectl config set-credentials ${env}-user --token=$(cat ${env}-token.txt)
  kubectl config set-context ${env}-context --cluster=my-cluster --user=${env}-user --namespace=${env}
done
```

### Step 11: Test access
Test access for each environment:
```bash
for env in dev test pre-prod prod; do
  kubectl --context=${env}-context get pods
done
```

## Part 2: Self-Managed Kubernetes Setup

### Step 1: Set up a Kubernetes cluster
Ensure you have a running Kubernetes cluster and kubectl configured to access it.

### Step 2: Create namespaces
```bash
kubectl create namespace dev
kubectl create namespace test
kubectl create namespace pre-prod
kubectl create namespace prod
```

### Step 3: Generate private keys for users
```bash
for env in dev test pre-prod prod; do
  openssl genrsa -out ${env}-user.key 2048
done
```

### Step 4: Create Certificate Signing Requests (CSRs)
```bash
for env in dev test pre-prod prod; do
  openssl req -new -key ${env}-user.key -out ${env}-user.csr -subj "/CN=${env}-user/O=${env}-group"
done
```

### Step 5: Sign the CSRs with the Kubernetes CA
Locate your Kubernetes CA certificate and key on the master node:
- CA Certificate: `/etc/kubernetes/pki/ca.crt`
- CA Key: `/etc/kubernetes/pki/ca.key`

Sign the CSRs:
```bash
for env in dev test pre-prod prod; do
  openssl x509 -req -in ${env}-user.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out ${env}-user.crt -days 365
done
```

### Step 6: Create Kubernetes RBAC resources
Create a file named `k8s-rbac.yaml`:

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: dev-role
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-role-binding
  namespace: dev
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: dev-role
  apiGroup: rbac.authorization.k8s.io
---
# Repeat similar Role and RoleBinding for test, pre-prod, and prod namespaces
# Adjust permissions as needed for each environment
```

Apply the RBAC configuration:
```bash
kubectl apply -f k8s-rbac.yaml
```

### Step 7: Configure kubectl for each user
```bash
for env in dev test pre-prod prod; do
  kubectl config set-credentials ${env}-user --client-certificate=${env}-user.crt --client-key=${env}-user.key
  kubectl config set-context ${env}-context --cluster=your-cluster-name --user=${env}-user --namespace=${env}
done
```
Replace `your-cluster-name` with the name of your Kubernetes cluster.

### Step 8: Test access
Test access for each environment:
```bash
for env in dev test pre-prod prod; do
  kubectl --context=${env}-context get pods
done
```

## Key Points for Interview Explanation

1. **Multi-Environment Setup**: 
   Explain that you've created four distinct namespaces (dev, test, pre-prod, prod) to isolate resources and permissions for different stages of the development lifecycle.

2. **Principle of Least Privilege**: 
   Highlight that you've created separate users/roles for each environment, allowing you to grant only the necessary permissions for each stage.

3. **AWS IAM Integration (for EKS)**:
   - Describe how you created IAM users and policies for each environment.
   - Explain the importance of the `aws-auth` ConfigMap in mapping IAM users to Kubernetes RBAC.
   - Mention the use of STS (Security Token Service) when getting tokens for kubectl.

4. **RBAC Configuration**:
   - Explain the concepts of Roles (namespace-specific permissions) and RoleBindings (associating users/groups with roles).
   - Highlight how you've used groups in EKS (`dev-group`, `test-group`, etc.) for easier management.

5. **Certificate Management (for self-managed)**:
   - Describe the process of creating and signing certificates for each user.
   - Emphasize the importance of secure handling of the Kubernetes CA key.

6. **kubectl Context**:
   Explain how you've set up separate kubectl contexts for each environment, allowing users to easily switch between them.

7. **Scalability and Maintenance**:
   Discuss how this setup allows for easy onboarding of new team members and adjustment of permissions as projects evolve.

8. **Security Considerations**:
   - Mention the importance of regular audits of RBAC configurations.
   - Discuss potential additional security measures like network policies or Pod Security Policies.

9. **Real-world Adaptations**:
   - Discuss how in a real production environment, you might integrate with an external identity provider.
   - Mention the potential use of tools like Terraform or CloudFormation for infrastructure-as-code management of these configurations.

10. **Differences between EKS and Self-Managed**:
    Highlight the key differences in setup and management, particularly around user authentication and certificate management.

Remember to emphasize that this setup provides a balance between security (isolation of environments) and usability (easy access for authorized users). Be prepared to discuss how you might adjust this setup for specific organizational needs or security requirements.
