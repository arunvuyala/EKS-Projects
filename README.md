# EKS-Projects with Fargate and ALB Ingress

I recently completed a POC to deploy a game application on an EKS cluster, using the AWS ALB Ingress Controller to expose the app externally.

### AWS Services Used:

* **EKS:** Managed Kubernetes service
* **Fargate:** Serverless compute for running pods
* **ECR:** Container image registry
* **CoreDNS:** Service discovery
* **kube-proxy:** Implements iptables for networking
* **AWS VPC CNI:** Pod networking integration
* **AWS IAM OIDC Provider:** Authentication for the cluster

### Tools:

* **AWS CLI:** AWS configuration and management
* **eksctl:** Create and manage EKS clusters
* **kubectl:** Kubernetes CLI
* **Helm:** Package manager to install ALB ingress controller

### High-Level Steps:

1. **Create EKS Cluster with Fargate:**

```bash
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```

2. **Update kubeconfig:**

```bash
aws eks update-kubeconfig --name demo-cluster --region us-east-1
```

3. **Create Fargate Profile (for specific namespace):**

```bash
eksctl create fargateprofile \
  --cluster demo-cluster \
  --region us-east-1 \
  --name alb-sample-app \
  --namespace game-2048
```

4. **Deploy the game application and ingress manifest:**

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

5. **Associate IAM OIDC Provider with the cluster:**

```bash
eksctl utils associate-iam-oidc-provider --cluster demo-cluster --approve
```

6. **Create IAM policy for ALB Controller:**

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

7. **Create Service Account with IAM role for ALB Controller:**

```bash
eksctl create iamserviceaccount \
  --cluster demo-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

8. **Install ALB Controller via Helm:**

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<vpc-id>
```

9. **Verify ALB Controller Deployment:**

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

10. **Access the Game Application:**
    Retrieve the Load Balancer DNS name:

```bash
kubectl get ingress -n game-2048
```

Open the DNS URL in your browser (allow 5-10 minutes for full provisioning):

```
http://k8s-game2048-ingress2-bcac0b5b37-1586363263.us-east-1.elb.amazonaws.com
```

Your game application should now be accessible!

---


