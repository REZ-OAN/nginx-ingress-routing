# Kubernetes Ingress Controller Setup

This document provides instructions on deploying a Kubernetes cluster with an Ingress Controller to expose frontend and backend services via domain names.

## Prerequisites

- A running Kubernetes cluster.
- `kubectl` installed and configured.
- A domain name (e.g., `example.com`) with DNS records pointing to your cluster's LoadBalancer IP.

## Resources Included

- **Frontend Deployment and Service:** Serves an NGINX frontend with a custom `index.html` file from a ConfigMap.
- **Backend Deployment and Service:** Runs a lightweight backend using Node.js to respond with a simple message.
- **Ingress Resource:** Configures routes for the frontend and backend services.
- **Ingress NGINX Controller:** Manages Ingress rules and serves as the entry point for HTTP and HTTPS traffic.

## Deployment Steps

### 1. Deploy the Frontend Service

The frontend deployment uses NGINX to serve a static HTML file defined in a ConfigMap.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: frontend
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: frontend-content
          mountPath: /usr/share/nginx/html
      volumes:
      - name: frontend-content
        configMap:
          name: frontend-content
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-content
data:
  index.html: |
    <html>
      <body>
        <h1>Welcome to the Frontend!</h1>
      </body>
    </html>
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

Apply the configuration:

```bash
kubectl apply -f frontend.yaml
```

### 2. Deploy the Backend Service

The backend deployment runs a simple Node.js application that listens on port 3000.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: backend
        image: node:18-alpine
        command: ["sh", "-c", "echo \"HTTP/1.1 200 OK\n\nHello from Backend!\" | nc -l -p 3000"]
        ports:
        - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
```

Apply the configuration:

```bash
kubectl apply -f backend.yaml
```

### 3. Deploy the Ingress Resource

The Ingress resource routes traffic to the frontend and backend services based on the host.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-demo
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 80
```

Apply the configuration:

```bash
kubectl apply -f ingress.yaml
```

### 4. Deploy the Ingress NGINX Controller

Deploy the NGINX Ingress Controller to manage Ingress resources.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: controller
        image: k8s.gcr.io/ingress-nginx/controller:v1.9.0
        args:
        - /nginx-ingress-controller
        - --configmap=$(POD_NAMESPACE)/nginx-configuration
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        ports:
        - name: http
          containerPort: 80
        - name: https
          containerPort: 443
---
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: ingress-nginx
  ports:
  - name: http
    port: 80
    targetPort: 80
  - name: https
    port: 443
    targetPort: 443
```

Apply the configuration:

```bash
kubectl apply -f ingress-nginx.yaml
```

### 5. Access the Services

- Update your DNS records to point `app.example.com` and `api.example.com` to the LoadBalancer IP of the Ingress NGINX Controller.
- Access the frontend at `http://app.example.com`.
- Access the backend at `http://api.example.com`.

## Verifying the Setup

1. Run the following command to get the LoadBalancer IP:

```bash
kubectl get svc -n ingress-nginx
```

2. Ensure DNS records point to this IP.
3. Test the frontend:

```bash
curl http://app.example.com
```

4. Test the backend:

```bash
curl http://api.example.com
```

## Cleanup

To delete all resources:

```bash
kubectl delete -f frontend.yaml
kubectl delete -f backend.yaml
kubectl delete -f ingress.yaml
kubectl delete -f ingress-nginx.yaml
```

