Got it ✅
Here’s a **professional all-in-one `README.md`** for your GitLab repository.

````markdown
# Harbor + K3s Integration Guide

This repository provides a step-by-step guide to integrating **Harbor** (container image registry) with **K3s** (lightweight Kubernetes).  
It ensures secure image pulling and pushing using SSL certificates and private registry authentication.

---

## 📌 Prerequisites

- A running **K3s cluster** (Linux environment)
- Installed **kubectl**
- Installed **Helm** (for Harbor installation)
- Domain name (e.g., `harbor.example.com`)
- SSL certificates (self-signed or from a trusted CA)

---

## ⚙️ 1. Install Harbor on K3s

### Using Helm

```bash
helm repo add harbor https://helm.goharbor.io
helm repo update

kubectl create namespace harbor

helm install harbor harbor/harbor \
  --namespace harbor \
  --set expose.ingress.hosts.core=harbor.example.com \
  --set expose.ingress.tls.secretName=harbor-tls
````

✅ Verify installation:

```bash
kubectl get pods -n harbor
```

Access Harbor UI at:
👉 `https://harbor.example.com`

---

## 🔑 2. Configure SSL Certificates

If using **self-signed certificates**:

```bash
openssl req -newkey rsa:4096 -nodes -sha256 -keyout harbor.key -x509 -days 365 -out harbor.crt
```

* Place certificates in `/etc/docker/certs.d/harbor.example.com/`
* Update K3s with certificates:

```bash
sudo mkdir -p /etc/rancher/k3s/harbor
sudo cp harbor.crt /etc/rancher/k3s/harbor/
sudo cp harbor.key /etc/rancher/k3s/harbor/
```

Restart K3s service:

```bash
sudo systemctl restart k3s
```

---

## 🔐 3. Authenticate with Harbor

### Docker Login

```bash
docker login harbor.example.com
```

### K3s Secret for Pulling Images

```bash
kubectl create secret docker-registry harbor-creds \
  --docker-server=harbor.example.com \
  --docker-username=admin \
  --docker-password=yourpassword \
  --namespace=default
```

Patch the default service account to use the secret:

```bash
kubectl patch serviceaccount default \
  -p '{"imagePullSecrets": [{"name": "harbor-creds"}]}'
```

---

## 📦 4. Push & Pull Images

### Push an Image

```bash
docker tag nginx:latest harbor.example.com/library/nginx:latest
docker push harbor.example.com/library/nginx:latest
```

### Deploy in K3s

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
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
        image: harbor.example.com/library/nginx:latest
        ports:
        - containerPort: 80
```

Apply:

```bash
kubectl apply -f nginx-deployment.yaml
```

---

## 🔍 5. Verify Deployment

Check pods:

```bash
kubectl get pods
```

Check logs:

```bash
kubectl logs -f <pod-name>
```

---

## ✅ Best Practices

* Always use **TLS/SSL** for Harbor
* Use **robot accounts** for CI/CD authentication
* Regularly rotate **credentials**
* Monitor Harbor logs for security issues
* Enable **content trust & vulnerability scanning**

---

## 📖 References

* [Harbor Documentation](https://goharbor.io/docs/)
* [K3s Documentation](https://rancher.com/docs/k3s/latest/en/)
* [Helm Harbor Chart](https://artifacthub.io/packages/helm/harbor/harbor)

---

## 👨‍💻 Author

**Jyotiprakash Nayak**
📍 IIT Kanpur | Electrical Engineering
🔗 [Codeforces](https://codeforces.com/profile/solver1)

```

---

Do you want me to also **add architecture diagrams** (Harbor <-> K3s flow) inside this README with Markdown image placeholders so it looks even more professional for GitLab?
```
