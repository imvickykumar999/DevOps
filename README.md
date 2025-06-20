# `DevOps`

![alt text](screenshots/image-1.png)
![alt text](screenshots/image.png)

# 🧰 Full DevOps Pipeline: Flask + Docker + K8s + Jenkins (Error-Proof Setup)

---

## 🔧 Prerequisites (Install Once on Ubuntu/Debian)

```bash
# 1. Update package list
sudo apt update

# 2. Install dependencies
sudo apt install -y apt-transport-https ca-certificates curl

# 3. Add the Google Cloud public signing key
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

# 4. Add Kubernetes repo
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list > /dev/null

# 5. Update apt again
sudo apt update

# 6. Install kubectl
sudo apt install -y kubectl

# 7. Install Docker
sudo apt install -y docker.io

# 8. Install Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

---

## ✅ Step 1: Create Project Directory

```bash
mkdir flask-k8s-demo && cd flask-k8s-demo
```

---

## ✅ Step 2: Flask App Code

**`app.py`**

```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello, Kubernetes!"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

---

## ✅ Step 3: Python Dependencies

**`requirements.txt`**

```txt
flask
```

---

## ✅ Step 4: Dockerfile

**`Dockerfile`**

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app.py .
EXPOSE 5000
CMD ["python", "app.py"]
```

---

## ✅ Step 5: Kubernetes YAMLs

### 📄 `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask
  template:
    metadata:
      labels:
        app: flask
    spec:
      containers:
      - name: flask
        image: localhost/flask-hello-k8s:latest   # 🔥 Key Fix: use localhost
        imagePullPolicy: Never                    # 🔥 Avoid remote pull
        ports:
        - containerPort: 5000
```

### 📄 `service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-service
spec:
  type: NodePort
  selector:
    app: flask
  ports:
  - port: 80
    targetPort: 5000
    nodePort: 30001
```

---

## ✅ Step 6: Start Minikube

```bash
minikube start
```

---

## ✅ Step 7: Build Docker Image (Inside Minikube)

```bash
# Switch to Minikube's Docker engine
eval $(minikube docker-env)

# Build with localhost prefix to prevent image pull errors
docker build -t localhost/flask-hello-k8s:latest .
```

---

## ✅ Step 8: Deploy to Kubernetes

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

Verify pods:

```bash
kubectl get pods
```

You should see:
`STATUS: Running`

---

## ✅ Step 9: Access Your App

```bash
minikube service flask-service
```

It will open:
**[http://192.168.49.2:30001](http://192.168.49.2:30001)**
➡️ Shows: `Hello, Kubernetes!`

---

## ✅ Step 10: Run Jenkins in Docker

```bash
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $HOME/.kube:/root/.kube \
  -v $HOME/.minikube:/root/.minikube \
  jenkins/jenkins:lts
```

---

## ✅ Step 11: Setup Jenkins

1. Open [http://localhost:8080](http://localhost:8080)
2. Unlock Jenkins:

   ```bash
   docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
   ```
3. Install Suggested Plugins
4. Create Admin User

---

## ✅ Step 12: Create Freestyle Job in Jenkins

1. Job name: `flask-k8s-deploy`
2. Choose: **Freestyle project**
3. Add **Build Step → Execute Shell**

### 🔧 Paste this script:

```bash
#!/bin/bash

# Use Docker inside Minikube
eval $(minikube docker-env)

# Build Docker image (with localhost tag)
docker build -t localhost/flask-hello-k8s:latest .

# Apply K8s manifests
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# Optional: Show URL
minikube service flask-service --url
```

---

## ✅ Common Fixes (Already Included)

| Problem                  | Fix                                                              |
| ------------------------ | ---------------------------------------------------------------- |
| `ErrImagePull`           | Use `imagePullPolicy: Never` & tag as `localhost/image-name:tag` |
| `SVC_UNREACHABLE`        | Ensure pods are `Running` and service matches deployment labels  |
| Docker image not visible | Use `eval $(minikube docker-env)` then `docker build`            |
| Jenkins can't see Docker | Mount `/var/run/docker.sock` and Minikube config into container  |

---

## 🏁 Done!

Your **end-to-end CI/CD pipeline is now ready**:

* Flask app served on Kubernetes ✅
* Image built locally with Docker ✅
* Managed and deployed through Jenkins ✅
* Served via Minikube NodePort ✅
* All 100% local and error-resilient ✅
