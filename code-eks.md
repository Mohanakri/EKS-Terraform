Awesome! Below is the complete GitHub-ready Markdown content for your Kubernetes setup — with all YAML code blocks correctly wrapped using triple backticks and formatted to render perfectly in .md files on GitHub.


---

✅ Save this as: eks-app-deployment.md

# 🚀 Kubernetes EKS Deployment with AWS Secrets Manager, ALB, EBS, and HPA

This guide defines Kubernetes manifests to deploy a secure, scalable application on AWS EKS using:

- ✅ AWS Secrets Manager via CSI driver
- ✅ EBS Persistent Volumes (gp3)
- ✅ ALB Ingress Controller for external access
- ✅ HorizontalPodAutoscaler for autoscaling
- ✅ NetworkPolicy for securing traffic
- ✅ ConfigMap for application configuration

---

## 1. 🛡️ ServiceAccount with IAM Role for Secrets Manager

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: secrets-manager-sa
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::YOUR-ACCOUNT-ID:role/EKSSecretsManagerRole


---

2. 🔐 SecretProviderClass for AWS Secrets Manager

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

3. 🚀 Deployment for Web App

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

4. 📦 PersistentVolumeClaim (PVC) for EBS

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

5. 🧱 StorageClass for gp3 EBS

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

6. 🛰️ ClusterIP Service for Internal Access

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

7. 🌐 Ingress with AWS ALB (External Access)

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:YOUR-REGION:YOUR-ACCOUNT-ID:certificate/YOUR-CERT-ID
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/load-balancer-name: my-web-app-alb
    alb.ingress.kubernetes.io/tags: Environment=production,Team=backend
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

8. 📈 HorizontalPodAutoscaler (HPA)

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

9. 🔒 Network Policy for Security

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

10. ⚙️ ConfigMap for App Settings

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


---

> ✅ Pro Tip: Store this file in your GitHub repo at docs/eks-app-deployment.md or manifests/README.md for easy documentation of your Kubernetes architecture.



---

If you'd like, I can export this as a `.md` file and give you a direct download link as well. Want that?

