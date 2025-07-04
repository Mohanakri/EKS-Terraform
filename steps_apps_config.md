Perfect — you're starting clean with Terraform for EKS setup. Based on your Terraform code and goals (EKS with EBS volumes and ALB Ingress for NGINX), I’ll now give you a **complete and step-by-step guide**:

---

## ✅ Step-by-Step: Full EKS + EBS + ALB Setup

---

### 🧱 1. Terraform EKS Infrastructure (✅ You Already Have It)

Your code provisions:

* VPC, 2 Subnets (public), Route Table
* Security Groups
* IAM Roles for Control Plane and Nodes
* EKS Cluster and Managed Node Group (v1.28)

**Double-check**:

* `var.ssh_key_name` must be defined.
* You’re using **public subnets**, good for ALB.

Apply it:

```bash
terraform init
terraform apply -auto-approve
```

---

### 🔍 2. Update `kubeconfig`

Once EKS is created:

```bash
aws eks update-kubeconfig \
  --name devopsshack-cluster \
  --region ap-south-1
```

Test it:

```bash
kubectl get nodes
```

---

## 🪛 3. Install EBS CSI Driver (for Persistent Storage)

### 1️⃣ Create IAM role for EBS CSI driver:

```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster devopsshack-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --role-name AmazonEKS_EBS_CSI_DriverRole
```

### 2️⃣ Install driver as EKS addon:

```bash
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster devopsshack-cluster \
  --service-account-role-arn arn:aws:iam::<your-account-id>:role/AmazonEKS_EBS_CSI_DriverRole \
  --force
```

---

## 📦 4. Create StorageClass and PVC

### 🔹 `storageclass.yaml`

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
```

### 🔹 `pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-ebs-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 1Gi
```

Apply:

```bash
kubectl apply -f storageclass.yaml
kubectl apply -f pvc.yaml
```

---

## 🧱 5. Deploy NGINX Using the EBS Volume

### 🔹 `nginx-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
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
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: ebs-volume
      volumes:
      - name: ebs-volume
        persistentVolumeClaim:
          claimName: nginx-ebs-pvc
```

Apply it:

```bash
kubectl apply -f nginx-deployment.yaml
kubectl get pods
```

---

## 🌐 6. Install AWS Load Balancer Controller

### 1️⃣ Add Helm repo

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

### 2️⃣ Associate OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
  --region ap-south-1 \
  --cluster devopsshack-cluster \
  --approve
```

### 3️⃣ Create IAM role + service account:

```bash
eksctl create iamserviceaccount \
  --cluster devopsshack-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::aws:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve \
  --override-existing-serviceaccounts \
  --region ap-south-1
```

### 4️⃣ Install with Helm

> 🔁 Replace `<your-vpc-id>` from Terraform output.

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=devopsshack-cluster \
  --set serviceAccount.create=false \
  --set region=ap-south-1 \
  --set vpcId=<your-vpc-id> \
  --set serviceAccount.name=aws-load-balancer-controller
```

Check:

```bash
kubectl get pods -n kube-system | grep aws-load
kubectl get endpoints -n kube-system aws-load-balancer-webhook-service
```

✅ Ensure pod is **Running** and webhook has **endpoints**.

---

## 🌍 7. Create NodePort Service + Ingress for NGINX

### 🔹 `nginx-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
  type: NodePort
```

```bash
kubectl apply -f nginx-service.yaml
```

---

### 🔹 `nginx-ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx
                port:
                  number: 80
```

```bash
kubectl apply -f nginx-ingress.yaml
```

Check:

```bash
kubectl get ingress
```

✅ It will return an ALB DNS — open in browser 🚀.

---

## 📁 Want All Files in One Repo?

Let me know and I’ll bundle:

* `storageclass.yaml`, `pvc.yaml`
* `nginx-deployment.yaml`, `nginx-service.yaml`, `nginx-ingress.yaml`
* Plus optional `alb-controller-install.sh`

Ready to continue? Want a ZIP or GitHub repo layout?
