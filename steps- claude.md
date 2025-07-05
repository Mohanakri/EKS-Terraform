I'll create a comprehensive deployment with AWS Secrets Manager, ingress, and load balancer for your EKS cluster.

# 1. AWS Secrets Manager Setup
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: secrets-manager-sa
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::YOUR-ACCOUNT-ID:role/EKSSecretsManagerRole

---
# 2. SecretProviderClass for AWS Secrets Manager
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: app-secrets
  namespace: default
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "my-app-secrets"
        objectType: "secretsmanager"
        jmesPath:
          - path: "database_url"
            objectAlias: "db-url"
          - path: "api_key"
            objectAlias: "api-key"
          - path: "jwt_secret"
            objectAlias: "jwt-secret"
  secretObjects:
    - secretName: app-secret-k8s
      type: Opaque
      data:
        - objectName: "db-url"
          key: "database_url"
        - objectName: "api-key"
          key: "api_key"
        - objectName: "jwt-secret"
          key: "jwt_secret"

---
# 3. Deployment with AWS Secrets Manager
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-web-app
  namespace: default
  labels:
    app: my-web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-web-app
  template:
    metadata:
      labels:
        app: my-web-app
    spec:
      serviceAccountName: secrets-manager-sa
      containers:
      - name: web-app
        image: nginx:latest
        ports:
        - containerPort: 80
          name: http
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secret-k8s
              key: database_url
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secret-k8s
              key: api_key
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: app-secret-k8s
              key: jwt_secret
        volumeMounts:
        - name: secrets-store
          mountPath: "/mnt/secrets"
          readOnly: true
        - name: ebs-storage
          mountPath: "/data"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: secrets-store
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: "app-secrets"
      - name: ebs-storage
        persistentVolumeClaim:
          claimName: web-app-pvc

---
# 4. PersistentVolumeClaim for EBS
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-app-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3-storage
  resources:
    requests:
      storage: 20Gi

---
# 5. StorageClass for EBS
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
  encrypted: "true"
  iops: "3000"
  throughput: "125"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true

---
# 6. Service for the Deployment
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
  namespace: default
  labels:
    app: my-web-app
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
  selector:
    app: my-web-app

---
# 7. Ingress with AWS Load Balancer Controller
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  namespace: default
  annotations:
    # AWS Load Balancer Controller annotations
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:YOUR-REGION:YOUR-ACCOUNT-ID:certificate/YOUR-CERT-ID
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/load-balancer-name: my-web-app-alb
    alb.ingress.kubernetes.io/tags: Environment=production,Team=backend
    # Health check settings
    alb.ingress.kubernetes.io/healthcheck-path: /
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '30'
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '5'
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-service
            port:
              number: 80
  - host: api.myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: web-app-service
            port:
              number: 80

---
# 8. HorizontalPodAutoscaler for scaling
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80

---
# 9. Network Policy for security
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-app-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: my-web-app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to: []
    ports:
    - protocol: TCP
      port: 443
    - protocol: TCP
      port: 80
    - protocol: UDP
      port: 53

---
# 10. ConfigMap for application configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-app-config
  namespace: default
data:
  app.properties: |
    server.port=80
    app.name=My Web Application
    app.version=1.0.0
    log.level=INFO
    cache.ttl=3600
  nginx.conf: |
    server {
        listen 80;
        server_name localhost;
        location / {
            root /usr/share/nginx/html;
            index index.html index.htm;
        }
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }

Now, here are the prerequisite setup commands you need to run before applying this configuration:

## Prerequisites Setup Commands

**1. Install AWS Secrets Store CSI Driver**
```bash
# Add the secrets store CSI driver
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --namespace kube-system

# Install AWS provider
kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml
```

**2. Create IAM Role for Secrets Manager**
```bash
# Create IAM policy for secrets manager
cat <<EOF > secrets-manager-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret"
            ],
            "Resource": "arn:aws:secretsmanager:*:*:secret:my-app-secrets-*"
        }
    ]
}
EOF

aws iam create-policy --policy-name EKSSecretsManagerPolicy --policy-document file://secrets-manager-policy.json

# Create service account with IAM role
eksctl create iamserviceaccount \
    --name secrets-manager-sa \
    --namespace default \
    --cluster YOUR-CLUSTER-NAME \
    --attach-policy-arn arn:aws:iam::YOUR-ACCOUNT-ID:policy/EKSSecretsManagerPolicy \
    --approve
```

**3. Create Secret in AWS Secrets Manager**
```bash
# Create the secret in AWS Secrets Manager
aws secretsmanager create-secret \
    --name my-app-secrets \
    --description "Application secrets for my web app" \
    --secret-string '{"database_url":"postgresql://user:pass@db:5432/mydb","api_key":"your-api-key-here","jwt_secret":"your-jwt-secret-here"}'
```

**4. Install AWS Load Balancer Controller**
```bash
# Create IAM policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.4/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json

# Create service account
eksctl create iamserviceaccount \
  --cluster=YOUR-CLUSTER-NAME \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::YOUR-ACCOUNT-ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Install the controller
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=YOUR-CLUSTER-NAME \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

**5. Install EBS CSI Driver**
```bash
# Create IAM role for EBS CSI driver
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster YOUR-CLUSTER-NAME \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve

# Install EBS CSI driver
aws eks create-addon \
  --cluster-name YOUR-CLUSTER-NAME \
  --addon-name aws-ebs-csi-driver
```

## Deployment Commands

**6. Apply the Configuration**
```bash
# Apply all resources
kubectl apply -f deployment.yaml

# Check deployment status
kubectl get deployments
kubectl get pods
kubectl get ingress
kubectl get pvc
kubectl get secrets
```

## Verification Commands

```bash
# Check if secrets are mounted
kubectl exec -it deployment/my-web-app -- ls -la /mnt/secrets

# Check environment variables
kubectl exec -it deployment/my-web-app -- printenv | grep -E "DATABASE_URL|API_KEY|JWT_SECRET"

# Check ingress and load balancer
kubectl get ingress web-app-ingress
kubectl describe ingress web-app-ingress

# Check HPA status
kubectl get hpa
```

## Important Notes

1. **Replace placeholders**: Update `YOUR-ACCOUNT-ID`, `YOUR-REGION`, `YOUR-CLUSTER-NAME`, and `YOUR-CERT-ID` with your actual values
2. **SSL Certificate**: You'll need to create an ACM certificate for HTTPS
3. **DNS**: Point your domain to the ALB DNS name after deployment
4. **Security Groups**: The ALB controller will automatically create security groups
5. **Monitoring**: Consider adding CloudWatch monitoring and logging

This configuration provides a production-ready setup with high availability, auto-scaling, security, and persistent storage.