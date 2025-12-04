Here is **Item 3 → A complete, production-ready `README.md`** for your **Flask + Jenkins + OpenShift CI/CD project**.

You can copy/paste directly into your GitHub repository.

---

# 📘 **README.md — Flask CI/CD with Jenkins & OpenShift**

## 🚀 Overview

This repository demonstrates a **fully automated CI/CD pipeline** where:

* A **Flask web application** is stored in GitHub
* **Jenkins**, running inside **OpenShift Sandbox**, executes the CI/CD pipeline
* The pipeline builds a container image using **OpenShift BuildConfig**
* The app is deployed to OpenShift using a **DeploymentConfig**
* A public HTTPS route exposes the application

The final deployed output:

```
Hello from Flask CI/CD on OpenShift via Jenkins!
```

---

# 🏗️ Architecture Diagram

```
Developer → GitHub → Jenkins Pipeline → OpenShift Build → Deployment → Route
```

(Full detailed architecture diagram available in documentation.)

---

# 📂 Repository Structure

```
flask-jenkins-openshift/
│
├── flask-app/
│   ├── app.py
│   ├── Dockerfile
│   └── requirements.txt
│
├── openshift/
│   ├── bc-flask-app.yaml        # BuildConfig + ImageStream
│   └── dc-flask-app.yaml        # DeploymentConfig + Service + Route
│
└── Jenkinsfile                  # Jenkins pipeline
```

---

# 🐍 Flask Application

`flask-app/app.py`:

```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def home():
    return "Hello from Flask CI/CD on OpenShift via Jenkins!"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```

---

# 🐋 Dockerfile

`flask-app/Dockerfile`:

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 8080
CMD ["python3", "app.py"]
```

---

# ☸️ OpenShift BuildConfig (ImageStream + Build)

`openshift/bc-flask-app.yaml`:

```yaml
apiVersion: v1
kind: ImageStream
metadata:
  name: flask-app
---
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: flask-app
spec:
  source:
    type: Binary
  strategy:
    type: Docker
  output:
    to:
      kind: ImageStreamTag
      name: flask-app:latest
```

---

# ☸️ OpenShift DeploymentConfig + Service + Route

`openshift/dc-flask-app.yaml`:

```yaml
apiVersion: apps.openshift.io/v1
kind: DeploymentConfig
metadata:
  name: flask-app
spec:
  replicas: 1
  selector:
    app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
      - name: flask-app
        image: "image-registry.openshift-image-registry.svc:5000/mg1982-dev/flask-app:latest"
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: flask-app
spec:
  selector:
    app: flask-app
  ports:
    - port: 8080
      targetPort: 8080
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: flask-app
spec:
  to:
    kind: Service
    name: flask-app
  port:
    targetPort: 8080
  tls:
    termination: edge
```

---

# 🔧 Jenkins Pipeline (Jenkinsfile)

This pipeline uses the **OpenShift OAuth token automatically injected** into the Jenkins pod.

```groovy
pipeline {
    agent any

    environment {
        APP_NAME    = "flask-app"
        OC_PROJECT  = "mg1982-dev"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                sh 'ls -R'
            }
        }

        stage('Quick Python Check') {
            steps {
                sh '''
                  if command -v python3 >/dev/null 2>&1; then
                    cd flask-app
                    python3 -m py_compile app.py
                  else
                    echo "python3 not found, skipping check"
                  fi
                '''
            }
        }

        stage('Login to OpenShift (OAuth Mode)') {
            steps {
                sh '''
                  echo "Logging into OpenShift using Jenkins pod token..."
                  oc login --token=$(oc whoami -t) --server=https://kubernetes.default.svc --insecure-skip-tls-verify
                  oc project ${OC_PROJECT}
                '''
            }
        }

        stage('Ensure BuildConfig exists') {
            steps {
                sh '''
                  oc apply -f openshift/bc-flask-app.yaml
                '''
            }
        }

        stage('Start OpenShift Build') {
            steps {
                sh '''
                  oc start-build ${APP_NAME} --from-dir=flask-app --follow --wait
                '''
            }
        }

        stage('Deploy Application') {
            steps {
                sh '''
                  oc apply -f openshift/dc-flask-app.yaml
                  oc rollout status dc/${APP_NAME} --timeout=180s
                '''
            }
        }
    }

    post {
        success {
            sh '''
              echo "App URL:"
              oc get route ${APP_NAME} -o jsonpath='{.spec.host}{"\\n"}'
            '''
        }
        failure {
            echo "Pipeline failed. Check logs."
        }
    }
}
```

---

# 🚀 How to Run CI/CD

### **1️⃣ Push code to GitHub**

```
git add .
git commit -m "Initial commit"
git push
```

### **2️⃣ Jenkins automatically runs pipeline**

You will see:

* Successful login to OpenShift
* Image built using BuildConfig
* Deployment rolled out
* Route created

### **3️⃣ Get the public URL**

Pipeline will output:

```
flask-app-mg1982-dev.apps.<cluster>.openshiftapps.com
```

Visit in browser:

```
Hello from Flask CI/CD on OpenShift via Jenkins!
```

---

# 🛠️ Troubleshooting

| Issue                           | Cause                                   | Fix                                                  |
| ------------------------------- | --------------------------------------- | ---------------------------------------------------- |
| Jenkins cannot find credentials | Using OAuth mode, no credentials needed | Remove `withCredentials`, use `oc whoami -t`         |
| BuildConfig not found           | YAML not applied                        | Ensure `oc apply -f openshift/bc-flask-app.yaml` ran |
| “ImagePullBackoff”              | Wrong ImageStream reference             | Ensure DeploymentConfig uses internal registry path  |
| Route gives 503                 | Pod not running / readiness failing     | Check logs: `oc logs -f dc/flask-app`                |

Full troubleshooting guide is provided separately.

---

# 🎉 Conclusion

You now have a fully working **CI/CD pipeline** using:

✓ GitHub
✓ Jenkins on OpenShift Sandbox
✓ OpenShift BuildConfig
✓ OpenShift DeploymentConfig
✓ Automated public HTTPS route

Your pipeline successfully delivers a real, production-ready CI/CD workflow.

---
