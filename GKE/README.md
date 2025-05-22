Objective of GKE hands-on lab :
--------

1. Create a GKE standard cluster with Workload Identity.
2. Create GCP and k8s Service Accounts.
3. Bind them via workload identity.
4. Deploy a pod that uses the binding to access GCS.

**Step by step guide**

* Enable Required APIs
```
gcloud services enable container.googleapis.com iam.googleapis.com compute.googleapis.com
```
* Create a GKE standard cluster with workload identity enabled

```
PROJECT_ID=$(gcloud config get-value project)
gcloud container clusters create demo-cluster \
  --zone us-central1-a \
  --num-nodes=2 \
  --workload-pool=$PROJECT_ID.svc.id.goog
```

* ssh/connect to the demo cluster

* Get Credentials for the cluster
```
gcloud container cluster get-credentials demo-cluster --zone us-central1-a
```

* Create a namespace and k8s service account
```
kubectl create namespace demo

kubectl create serviceaccount k8s-app-sa --namespace demo
```

* Create a GCP Service Account and grant access to GCS
```
gcloud iam service-accounts create gcs-access-sa \
  --display-name "GCS Access Service Account"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:gcs-access-sa@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

```
* Bind k8s SA to GCP SA
```
gcloud iam service-accounts add-iam-policy-binding gcs-access-sa@$PROJECT_ID.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:$PROJECT_ID.svc.id.goog[demo/k8s-app-sa]"

kubectl annotate serviceaccount k8s-app-sa \
  --namespace demo \
  iam.gke.io/gcp-service-account=gcs-access-sa@$PROJECT_ID.iam.gserviceaccount.com

```

* Deploy pod to use Workload Identity
```
apiVersion: v1
kind: Pod
metadata:
  name: gcs-test
  namespace: demo
spec:
  serviceAccountName: k8s-app-sa
  containers:
  - name: gcs
    image: google/cloud-sdk
    command: ["/bin/sh", "-c"]
    args: ["gsutil ls gs://cloud-build-artifacts/"]
```

* Apply the pod yaml file
```
kubectl apply -f <pod-file>
```
* Verify
```
kubectl logs gcs-test -n demo
```

GCP Services Used :
------

**1. *GKE* :** 

* Types : 

    * GKE Standard : GCP manages the Control Planes, but the nodes are managed by user.

        *   Best for :  full flexibility, ideal for production.

    * GKE Autopilot : Fully managed by GCP. No access to nodes. 

         *   Best for : Hands-off operations, dev/test, fast launch

    * GKE Private : Nodes have no public IPs. API server access is restricted.

        *   Best for : High-security workloads

    * GKE Zonal/Regional : Better HA
        
        * Zonal : Single zone.
        * Regional : 3 zones + HA control plane.

* GKE Architecture :

  Control Plane (Managed by Google)
    * Kubernetes API server
    * Scheduler
    * Controller manager
    * etcd (backed up and replicated)
    * Cloud integrations (IAM, logging, monitoring)

  Node Pool (Managed by you or GCP)
    
    * Group of Compute Engine VMs
    * Each node runs:
        
        * kubelet
        * kube-proxy
        * Container runtime (containerd)

* GKE Autopilot V/s Standard

| Feature         | Autopilot              | Standard                   |
| --------------- | ---------------------- | -------------------------- |
| Node management | Fully managed          | User-managed               |
| Access to nodes | ❌ No (abstracted)      | ✅ Yes                      |
| Billing         | Per pod (vCPU, memory) | Per node (VM)              |
| Use case        | Simple, hands-off      | Advanced, custom workloads |
| Security        | Hardened by default    | You must secure nodes      |
| GPU/TPU support | Limited                | Full support               |

* GKE Security Concepts : ??

**2. *Service Account* :** Special Google Cloud identity **(non-human or robot entity)** used by application, virtual machines, containers or other **GCP services to authenticate and interact with Google Cloud APIs** without human intervention.


* User-managed : Created and managed manually by user, roles assigned and SA impersonation controlled by user.

    * Use for : GKE workload Identity, CI/CD pipelines and Automaton scripts.

    * Google-managed : GCP auto-creates these for internal service use. User don't have any control on this. 

        * Use for : GCP-managed backups, Cloud Storage transfer service, etc.

    * default : Auto-created per project (eg; Compute Engine default SA with broad permission like editor).

        * Use : When you don't specify an SA for GCE VMs, App Engine, Cloud Functions.


**3. *Workload Identity* :** It Allows k8s workload (pods) in GKE to securely access Google Cloud services (like GCS, BigQuery, PUB/SUB, etc.) without needing to manage the  service account keys manually.

* *Traditionally (without WI)* : If your pod need access to GCP Services, you would download a SA key JSON file. Mount it into the pod and use it to authenticate.

    * *What problem does it resolve* : Prevent keys from being accidentally exposed (as you don't have to download SA keys).

    * *How* : By linking KSA to GSA (IAM policy binding allows KSA to impersonate GSA using annotation)

    * *Uses* : Deploy your workload using KSA.

        * When pod runs it uses **Google's metadata server and Workload Identity Federation** to obtain temporary credentials to acess GCP services.
    

