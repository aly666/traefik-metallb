# Project Traefik-MetalLB

<p align="center">
  <img src="images/diagram.png" alt="diagram" width="400" />
</p>

# Deploy Go API dengan Traefik IngressRoute dan Middleware CORS

## Prasyarat

- Kubernetes cluster (misal k3s, kind, minikube)
- `kubectl` sudah terhubung ke cluster
- Helm sudah terinstall
- Domain dapat dikelola DNS-nya (contoh `sutrisno.xyz`)
- Traefik sudah diinstall dengan CRD dan IngressRoute aktif

---

## 1. Install MetalLB (LoadBalancer untuk bare-metal)

```bash
kubectl create namespace metallb-system

helm repo add metallb https://metallb.github.io/metallb
helm repo update

helm install metallb metallb/metallb -n metallb-system
```

### Konfigurasi IP Pool MetalLB (sesuaikan IP sesuai jaringan lokal)

Buat file `metallb-config.yaml`:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2advertisement
  namespace: metallb-system
```

Apply konfigurasi:

```bash
kubectl apply -f metallb-config.yaml
```

---

## 2. Install Traefik dengan Helm (CRD dan IngressRoute aktif)

Buat file `traefik-values.yaml`:

```yaml
deployment:
  enabled: true

service:
  type: LoadBalancer

ingressClass:
  enabled: true
  isDefaultClass: true
  name: traefik

ports:
  web:
    port: 80
    expose:
      default: true
    protocol: TCP

  websecure:
    port: 443
    expose:
      default: true
    protocol: TCP

providers:
  kubernetesCRD: {}
  kubernetesIngress: {}

additionalArguments:
  - "--api.insecure=true"
  - "--api.dashboard=true"
  - "--entryPoints.web.address=:80"
  - "--entryPoints.websecure.address=:443"
```

Install Traefik:

```bash
kubectl create namespace traefik

helm repo add traefik https://traefik.github.io/charts
helm repo update

helm install traefik traefik/traefik -n traefik -f traefik-values.yaml
```

---

## 3. Atur DNS `app.sutrisno.xyz` ke IP LoadBalancer Traefik

Cek EXTERNAL-IP Traefik:

```bash
kubectl get svc -n traefik
```

Buat A record di DNS:

| Host            | Type | Value          |
|-----------------|------|----------------|
| app.sutrisno.xyz| A    | <EXTERNAL-IP>  |

---

## 4. Deploy MySQL dan Go API

Buat file `app-deploy.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: password
        - name: MYSQL_DATABASE
          value: go_crud
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
spec:
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
  clusterIP: None
---
apiVersion: v1
kind: Service
metadata:
  name: go-pasien-service
spec:
  selector:
    app: go-pasien-app
  ports:
  - port: 80
    targetPort: 3000
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-pasien-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: go-pasien-app
  template:
    metadata:
      labels:
        app: go-pasien-app
    spec:
      containers:
      - name: go-pasien-app
        image: aly666/simple-crud-app
        ports:
        - containerPort: 3000
        env:
        - name: DB_HOST
          value: mysql-service
        - name: DB_PORT
          value: "3306"
        - name: DB_USER
          value: root
        - name: DB_PASS
          value: password
        - name: DB_NAME
          value: go_crud
```

Deploy:

```bash
kubectl apply -f app-deploy.yaml
```

---

## 5. Buat Middleware CORS

File `middleware-cors.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: cors-middleware
  namespace: default
spec:
  headers:
    accessControlAllowOriginList:
      - "*"
    accessControlAllowMethods:
      - GET
      - POST
      - PUT
      - DELETE
      - OPTIONS
    accessControlAllowHeaders:
      - Content-Type
      - Authorization
      - X-Requested-With
    accessControlAllowCredentials: true
    addVaryHeader: true
```

Apply:

```bash
kubectl apply -f middleware-cors.yaml
```

---

## 6. Buat IngressRoute dengan Middleware CORS

File `ingressroute.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: go-pasien-ingress
  namespace: default
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`app.sutrisno.xyz`) && PathPrefix(`/api/pasien`)
      kind: Rule
      middlewares:
        - name: cors-middleware
          namespace: default
      services:
        - name: go-pasien-service
          port: 80
```

Apply:

```bash
kubectl apply -f ingressroute.yaml
```

---

## 7. Verifikasi

```bash
kubectl get pods,svc,ingressroute,middleware -A
```

Cek aplikasi di browser:

```
http://app.sutrisno.xyz/api/pasien
```

# treafik-metallb
