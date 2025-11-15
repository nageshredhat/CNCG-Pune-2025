# CNCG-Pune-2025


# **KServe, Cert-Manager, KEDA & Monitoring Stack Installation Guide**

This guide documents the installation of **KServe v0.15.0**, required CRDs, **cert-manager**, **KEDA**, and a basic **Prometheus monitoring stack**, including notes for GPU/DCGM and dashboards.  
All commands were verified in a Kubernetes cluster (example deployment timestamps omitted).

----------

## **Prerequisites**

Before installing any components, ensure you have:

-   A running Kubernetes cluster (v1.25+ recommended)
    
-   `kubectl` installed and configured
    
-   `helm` v3.11+ installed
    
-   Sufficient privileges to install CRDs and cluster-wide resources
    

----------

# ---------------------------------------------------------

# **1. Install KServe CRDs**

# ---------------------------------------------------------

KServe requires its CRDs to be installed separately prior to deploying controllers.

`helm upgrade --install kserve-crd oci://ghcr.io/kserve/charts/kserve-crd \
  --version v0.15.0 \
  -n kserve --create-namespace` 

Verify installation:

`helm ls -n kserve
kubectl get crd | grep kserve` 

Expected CRDs include:

-   `inferenceservices.serving.kserve.io`
    
-   `servingruntimes.serving.kserve.io`
    
-   `clusterservingruntimes.serving.kserve.io`
    
-   Others listed in installation logs
    

----------

# ---------------------------------------------------------

# **2. Install cert-manager**

# ---------------------------------------------------------

cert-manager is required for TLS certificates used by KServe’s webhook & components.

`helm repo add jetstack https://charts.jetstack.io --force-update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.19.1 \
  --set crds.enabled=true` 

Validate installation:

`kubectl get all -n cert-manager` 

You should see:

-   `cert-manager`
    
-   `cert-manager-cainjector`
    
-   `cert-manager-webhook`
    

All in **Running** state.

----------

# ---------------------------------------------------------

# **3. Install KServe Controller**

# ---------------------------------------------------------

### **Option A — RawDeployment mode**

`helm upgrade --install kserve oci://ghcr.io/kserve/charts/kserve \
  --version v0.15.0 \
  --set kserve.controller.deploymentMode=RawDeployment \
  -n kserve` 

### **Option B — Serverless mode**

(Requires Knative Serving)

`helm upgrade --install kserve oci://ghcr.io/kserve/charts/kserve \
  --version v0.15.0 \
  --set kserve.controller.deploymentMode=Serverless \
  -n kserve` 

### **Option C — Knative mode**

`helm upgrade --install kserve oci://ghcr.io/kserve/charts/kserve \
  --version v0.15.0 \
  --set kserve.controller.deploymentMode=Knative \
  -n kserve` 

Verify:

`kubectl get all -n kserve` 

Expected resources:

-   `kserve-controller-manager`
    
-   `kserve-webhook-server-service`
    

----------

# ---------------------------------------------------------

# **4. Install KEDA (for autoscaling)**

# ---------------------------------------------------------

`helm repo add kedacore https://kedacore.github.io/charts
helm repo update

helm install keda kedacore/keda \
  --namespace keda --create-namespace` 

Validate:

`kubectl get pods -n keda` 

Expected pods:

-   `keda-operator`
    
-   `keda-admission-webhooks`
    
-   `keda-operator-metrics-apiserver`
    

----------

# ---------------------------------------------------------

# **5. Install Prometheus Monitoring**

# ---------------------------------------------------------

Create namespace:

`kubectl create ns monitoring` 

Apply Prometheus manifest:

`kubectl apply -f prometheus.yaml -n monitoring` 

Check status:

`kubectl get all -n monitoring` 

Port-forward Prometheus UI:

`kubectl port-forward svc/prometheus-service 8080:8080 -n monitoring` 

Open in browser:

`http://localhost:8080` 

----------

# ---------------------------------------------------------

# **6. Add vLLM Dashboards, HPA Metrics & GPU (DCGM) Exporter**

# ---------------------------------------------------------

### **Install NVIDIA DCGM exporter**

If your cluster has NVIDIA GPUs:

`helm repo add nvidia https://nvidia.github.io/dcgm-exporter/helm-charts
helm repo update

helm install dcgm-exporter nvidia/dcgm-exporter \
  --namespace monitoring` 

Metrics become available at Prometheus endpoint.

