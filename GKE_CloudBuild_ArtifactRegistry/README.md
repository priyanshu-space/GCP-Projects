Q/A
----

* Cloud Build : 

    * 

Hands-on Lab
------------

**Objectives**

* Create a VPC native GKE cluster.
* Create a private Docker repo (Artifact Registry)
* Build and push a docker image using Cloud Build.
* Deploy that image to GKE via Kubernetes.

**Step by step guide**

* Enable Required APIs

```
gcloud services enable \
  container.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com
```

* Create a VPC native GKE cluster

```
PROJECT_ID=$(gcloud config get-value project)

gcloud container clusters create gke-net-demo \
  --zone us-central1-a \
  --enable-ip-alias \
  --num-nodes=2
```

* Create Artifact Registry

```
gcloud artifacts repositories create demo-repo \
  --repository-format=docker \
  --location=us-central1 \
  --description="Docker repo for GKE"
```

* Prepare a sample Docker app

```
mkdir hello-app && cd hello-app

cat <<EOF > app.py
from flask import Flask
app = Flask(__name__)
@app.route('/')
def hello():
    return "Hello from GKE with Cloud Build!"
EOF

cat <<EOF > Dockerfile
FROM python:3.9
WORKDIR /app
COPY app.py .
RUN pip install flask
CMD ["python", "app.py"]
EOF
```

* Create cloudbuild.yaml 

```
# file: cloudbuild.yaml
steps:
- name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', 'us-central1-docker.pkg.dev/$PROJECT_ID/demo-repo/hello-app', '.']
images:
- 'us-central1-docker.pkg.dev/$PROJECT_ID/demo-repo/hello-app'
```

* Submit build to cloud build : 

```
gcloud builds submit --region=us-central1 --config cloudbuild.yaml
```

* Deploy to GKE

```
gcloud container clusters get-credentials gke-net-demo --zone us-central1-a
```

* Create k8s deployment 

```
# file: deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: hello
        image: us-central1-docker.pkg.dev/<PROJECT-ID>/demo-repo/hello-app
        ports:
        - containerPort: 5000
```

* Verify extIP