# ingress-app
This README provides instructions for setting up and configuring the Nginx Ingress Controller and testing the application with Kind on Kubernetes.

## Lab: Setting up Nginx Ingress Controller on Kubernetes with Kind

### Step 1: Install the Nginx Ingress Controller

1. Visit the Nginx Ingress Controller documentation:
   https://kind.sigs.k8s.io/docs/user/ingress/

2. To install the Nginx Ingress Controller, ensure the following:
   - Expose ports 80 and 443 for HTTP and HTTPS traffic from outside the cluster.
   - Label the control plane (or master node) to prepare it for ingress setup.

### Delete Existing Cluster (if any)

```bash
kind delete cluster
```

### Create a New Cluster with Custom Configuration
```bash
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
EOF
```

### Wait for the Cluster to be Ready and Install the Nginx Ingress Controller
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

## Application Setup
The application has three endpoints:
- **Admin**: `www.example.com/admin`
- **API**: `api.example.com`
- **Main**: `www.example.com`

We’ll set up three Pods and three Services, followed by creating an Ingress object.

### Creating Pod and Service YAML Files
- **www Pod and Service**:
  - `www.yaml`:
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: www
      labels:
        app: www
    spec:
      containers:
      - name: www
        image: nginx
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: www
    spec:
      selector:
        app: www
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
    ```

- **api Pod and Service**:
  - `api.yaml`:
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: api
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: ealen/echo-server:latest
        ports:
        - containerPort: 80
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: api
    spec:
      selector:
        app: api
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
    ```

- **admin Pod and Service**:
  - `admin.yaml`:
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: admin
      labels:
        app: admin
    spec:
      containers:
      - name: admin
        image: ealen/echo-server:latest
        env:
        - name: ECHO_SERVER_BASE_PATH
          value: "/admin"
        ports:
        - containerPort: 80
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: admin
    spec:
      selector:
        app: admin
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
    ```

### Setting up the Ingress Object
- `ingress.yaml`:
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: ingress
  spec:
    ingressClassName: nginx
    rules:
    - host: www.example.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: www
              port:
                number: 80
        - path: /admin
          pathType: Exact
          backend:
            service:
              name: admin
              port:
                number: 80
    - host: api.example.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: api
              port:
                number: 80
  ```

### Verify Pods, Services, and Ingress Status
```bash
kubectl get pods
kubectl get svc
kubectl get ing
```

## Host Configuration
Add the following entries to `/etc/hosts` to route the domains locally:
```
127.0.0.1 api.example.com
127.0.0.1 www.example.com
```

## Testing the Application
Test the application by visiting:
- `www.example.com`
- `www.example.com/admin`
- `api.example.com`

> Note: Only `www.example.com/admin` will respond to exact matches, while `api.example.com/something` will respond due to the `Prefix` path type.
```