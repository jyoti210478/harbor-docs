# Harbor Installation and Integration with K3s Cluster

This document provides a **step-by-step guide** for installing Harbor (both online and offline installers) in a Virtual Machine (VM) and integrating it with a K3s Kubernetes cluster.

Harbor is an open-source **cloud-native registry** that secures artifacts with policies and role-based access control. It also supports image replication, vulnerability scanning, and multi-tenancy.

---

## 📌 Table of Contents

1. [Prerequisites](##prerequisites)
2. [Harbor Installation](##haharbor-installation)

   * [Online Installer](###online-installer)
   * [Offline Installer](###offline-installer)
3. [Generating Certificates](##generating-certificates)

   * [Generate a CA Certificate](###1-generate-a-ca-certificate)
   * [Generate a Server Certificate](###2-generate-a-server-certificate)
4. [Configuring Harbor](##configuring-harbor)
5. [Preparing & Installing Harbor](##preparing--installing-harbor)
6. [Verifying Installation](##verifying-installation)
7. [Integrating Harbor with K3s](##integrating-harbor-with-k3s)
8. [Pushing Images from K3s to Harbor](##pushing-images-from-k3s-to-harbor)
9. [Pulling Images from Harbor to K3s](##pulling-images-from-harbor-to-k3s)
10. [Example Deployment](##example-deployment)

---

## ✅ Prerequisites

* A VM with Docker installed.
* Root/sudo access.
* K3s cluster installed and running.
* Domain name resolution configured (`/etc/hosts`).

---

## 🚀 Harbor Installation

### Online Installer

```bash
wget https://github.com/goharbor/harbor/releases/download/v2.10.0/harbor-online-installer-v2.10.0.tgz
tar xvf harbor-online-installer-v2.10.0.tgz
cd harbor
```

### Offline Installer

```bash
wget https://github.com/goharbor/harbor/releases/download/v2.13.2/harbor-offline-installer-v2.13.2.tgz
tar xvf harbor-offline-installer-v2.13.2.tgz
cd harbor
```

---

## 🔐 Generating Certificates

### 1. Generate a CA Certificate

* Generate the **CA private key**:

```bash
openssl genrsa -out ca.key 4096
```

* Generate the **CA certificate**:

```bash
openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/C=IN/ST=Utter Pradesh/L=Kanpur/O=c3ihub/OU=Devops/CN=harbor.k8.c3ihub" \
 -key ca.key \
 -out ca.crt
```

### 2. Generate a Server Certificate

* Generate the **server private key**:

```bash
openssl genrsa -out harbor.k8.c3ihub.key 4096
```

* Generate the **CSR (Certificate Signing Request)**:

```bash
openssl req -sha512 -new \
 -subj "/C=CN/ST=Utter Pradesh/L=Kanpur/O=c3ihub/OU=Devops/CN=harbor.k8.c3ihub" \
 -key harbor.k8.c3ihub.key \
 -out harbor.k8.c3ihub.csr
```

* Create an **x509 v3 extension file**:

```bash
cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=harbor.k8.c3ihub
EOF
```

* Generate the **server certificate** using `v3.ext`:

```bash
openssl x509 -req -sha512 -days 3650 \
 -extfile v3.ext \
 -CA ca.crt -CAkey ca.key -CAcreateserial \
 -in harbor.k8.c3ihub.csr \
 -out harbor.k8.c3ihub.crt
```

* Place the certificates in Harbor directory:

```bash
sudo mkdir -p /etc/harbor/certs
sudo cp harbor.k8.c3ihub.crt harbor.k8.c3ihub.key ca.crt /etc/harbor/certs/
```

---

## ⚙️ Configuring Harbor

Edit Harbor configuration file:

```bash
cd harbor
nano harbor.yml.tmpl
```

Update the following values:

```yaml
hostname: harbor.k8.c3ihub
https:
  port: 443
  certificate: /etc/harbor/certs/harbor.k8.c3ihub.crt
  private_key: /etc/harbor/certs/harbor.k8.c3ihub.key
```

Save and copy the file:

```bash
cp harbor.yml.tmpl harbor.yml
```

---

## 🛠️ Preparing & Installing Harbor

* Prepare Harbor:

```bash
sudo ./prepare
```

* **Offline installer only** – load images:

```bash
sudo docker load -i harbor.v2.13.2.tar.gz
```

* Install Harbor:

```bash
sudo ./install.sh
```

---

## 🔎 Verifying Installation

Check running containers:

```bash
sudo docker ps
```

If Harbor is running, you should see multiple containers such as `harbor-core`, `harbor-portal`, `nginx`, etc.

---

## 🔗 Integrating Harbor with K3s

1. Copy the Harbor CA certificate to K3s cluster nodes:

```bash
sudo mkdir -p /etc/docker/certs.d/harbor.k8.c3ihub/
sudo cp ca.crt /etc/docker/certs.d/harbor.k8.c3ihub/
```

2. Edit `/etc/hosts` on each K3s node to resolve the Harbor domain:

```
<Harbor-VM-IP>   harbor.k8.c3ihub
```

---

## 📤 Pushing Images from K3s to Harbor

1. Login to Harbor:

```bash
sudo docker login harbor.k8.c3ihub
```

2. Tag your image:

```bash
sudo docker tag harbor.test.c3ihub/asset/am-react-frontend:16fc1436 \
harbor.k8.c3ihub/test/am-react-frontend:16fc1436
```

3. Push to Harbor:

```bash
sudo docker push harbor.k8.c3ihub/test/am-react-frontend:16fc1436
```

---

## 📥 Pulling Images from Harbor to K3s

### Step 1: Test Access

```bash
docker login harbor.k8.c3ihub
docker pull harbor.k8.c3ihub/test/am-react-frontend:16fc1436
```

* ✅ If successful → proceed.
* ❌ If unauthorized → configure credentials.

### Step 2: Create Kubernetes Secret (for private projects)

```bash
kubectl create secret docker-registry harbor-secret \
  --docker-server=harbor.k8.c3ihub \
  --docker-username=admin \
  --docker-password='<your-admin-password>' \
  --docker-email='you@example.com'
```

### Step 3: Use Secret in Deployment

Add the secret under `imagePullSecrets` in your Pod/Deployment manifest.

---

## 📦 Example Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: harbor.k8.c3ihub/test/am-react-frontend:16fc1436
      imagePullSecrets:
      - name: harbor-secret
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

---

## 🎯 Conclusion

You have successfully:

* Installed Harbor (online/offline installer).
* Configured SSL certificates.
* Integrated Harbor with a K3s cluster.
* Pushed and pulled images between Harbor and K3s.

Harbor now acts as a **secure private registry** for your Kubernetes workloads.

---

Would you like me to also **add a diagram** (architecture flow: VM with Harbor ⇄ K3s cluster ⇄ workloads) to make this README even more professional for GitLab?
