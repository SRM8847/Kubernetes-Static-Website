# Kubernetes-Static-Website

# 0) Prerequisites

* A working Kubernetes cluster + `kubectl` pointed at it

  ```bash
  kubectl cluster-info
  kubectl get nodes
  ```
* For local: **minikube** is easiest.
* For cloud: ensure your account/project & networking allow a public LoadBalancer.

We’ll work in a namespace called `web`.

```bash
kubectl create namespace web
```

---

# 1) Prepare your static site

Create a folder for your site (example content shown):

```
site/
├── index.html
└── styles.css
```

**index.html**

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8" />
  <title>My Static Site</title>
  <link rel="stylesheet" href="/styles.css" />
</head>
<body>
  <h1>Hello from Kubernetes 🚀</h1>
  <p>If you see this, your static site is live.</p>
</body>
</html>
```

**styles.css**

```css
body { font-family: system-ui, sans-serif; margin: 3rem; }
h1 { font-size: 2rem; }
```

---

# 2) Create a ConfigMap from your files

This loads each file into a ConfigMap (one key per file). It’s perfect for small sites and demos.

```bash
kubectl -n web create configmap static-site --from-file=site/
```

(If you prefer YAML in git, generate it once:)

```bash
kubectl -n web create configmap static-site --from-file=site/ \
  --dry-run=client -o yaml > 01-configmap.yaml
kubectl apply -f 01-configmap.yaml
```

---

# 3) Create the Deployment (nginx + mounted files)

Save as **02-deploy.yaml**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: static-site
  namespace: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: static-site
  template:
    metadata:
      labels:
        app: static-site
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
          volumeMounts:
            - name: site
              mountPath: /usr/share/nginx/html
              readOnly: true
      volumes:
        - name: site
          configMap:
            name: static-site
```

Apply it:

```bash
kubectl apply -f 02-deploy.yaml
kubectl -n web rollout status deploy/static-site
kubectl -n web get pods -o wide
```

---

# 4A) Expose it (Local cluster) — NodePort

Save as **03-svc-nodeport.yaml**:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: static-site
  namespace: web
spec:
  type: NodePort
  selector:
    app: static-site
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      nodePort: 30080   # optional; omit to let K8s choose in 30000–32767
```

Apply and test:

```bash
kubectl apply -f 03-svc-nodeport.yaml
kubectl -n web get svc static-site
```

* **minikube** quick open:

  ```bash
  minikube service -n web static-site --url
  ```

  Open the printed URL in your browser.

* **Without minikube helper:** browse to `http://<node-ip>:30080`
  (Get a node IP with `kubectl get nodes -o wide`.)

---

# 4B) Expose it (Cloud cluster) — LoadBalancer

Save as **03-svc-lb.yaml**:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: static-site
  namespace: web
spec:
  type: LoadBalancer
  selector:
    app: static-site
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
```

Apply and wait for the external IP:

```bash
kubectl apply -f 03-svc-lb.yaml
watch -n 2 kubectl -n web get svc static-site
```

Once `EXTERNAL-IP` shows up, open `http://<EXTERNAL-IP>/`.

---
