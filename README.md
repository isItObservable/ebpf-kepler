# Is it Observable
<p align="center"><img src="/image/logo.png" width="40%" alt="Is It observable Logo" /></p>

## Episode : What is Ebpf - tutorial using Kepler
This repository contains the files utilized during the tutorial presented in the dedicated IsItObservable episode related to ebpf.
<p align="center"><img src="/image/ebpf.png" width="40%" alt="ebpf Logo" /></p>

What you will learn
* How to use the [Kepler](https://sustainable-computing.io/html/index.html)

This repository showcase the usage of Kepler  with :
* The Otel-demo
* The OpenTelemetry Operator
* Nginx ingress controller
* Prometheus
* Grafana
* Dynatrace

We will send the kepler metrics to dynatrace and build a short dashboard.

## Prerequisite
The following tools need to be install on your machine :
- jq
- kubectl
- git
- gcloud ( if you are using GKE)
- Helm


## Deployment Steps in GCP

You will first need a Kubernetes cluster with 2 Nodes.
You can either deploy on Minikube or K3s or follow the instructions to create GKE cluster:
### 1.Create a Google Cloud Platform Project
```
PROJECT_ID="<your-project-id>"
gcloud services enable container.googleapis.com --project ${PROJECT_ID}
gcloud services enable monitoring.googleapis.com \
    cloudtrace.googleapis.com \
    clouddebugger.googleapis.com \
    cloudprofiler.googleapis.com \
    --project ${PROJECT_ID}
```
### 2.Create a GKE cluster
```
ZONE=europe-west3-a
NAME=isitobservable-ebpf
gcloud container clusters create "${NAME}" \
 --zone ${ZONE} --machine-type=e2-standard-2 --num-nodes=4
```
### 3.Clone the Github Repository
```
https://github.com/isItObservable/ebpf-exporter
cd ebpf-exporter
```
### 4.Deploy Nginx Ingress Controller
```
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```
#### 1. get the ip adress of the ingress gateway
Since we are using Ingress controller to route the traffic , we will need to get the public ip adress of our ingress.
With the public ip , we would be able to update the deployment of the ingress for :
* otel-demo
```
IP=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -ojson | jq -j '.status.loadBalancer.ingress[].ip')
```
#### 2. Update the manifest files

update the following files to update the ingress definitions :
```
sed -i "s,IP_TO_REPLACE,$IP," kubernetes-manifests/K8sdemo.yaml
sed -i "s,IP_TO_REPLACE,$IP," grafana/ingress.yaml
```
### 5.Prometheus

#### 1. Deploy Prometheus without Grafana
 ```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack  --set kubeStateMetrics.enabled=false --set nodeExporter.enabled=false
```
#### 2. Deploy the grafana ingress
```
kubectl apply -f grafana/ingress.yaml 
```
### 6. Dynatrace
#### 1. Dynatrace Tenant - start a trial
If you don't have any Dyntrace tenant , then i suggest to create a trial using the following link : [Dynatrace Trial](https://bit.ly/3KxWDvY)
Once you have your Tenant save the Dynatrace (including https) tenant URL in the variable `DT_TENANT_URL` (for example : https://dedededfrf.live.dynatrace.com)
```
DT_TENANT_URL=<YOUR TENANT URL>
```

##### Generate Token to ingest data
Create a Dynatrace token with the following scope:
* ingest metrics
* ingest OpenTelemetry traces
<p align="center"><img src="/image/data_ingest.png" width="40%" alt="data token" /></p>
Save the value of the token . We will use it later to store in a k8S secret

```
DATA_INGEST_TOKEN=<YOUR TOKEN VALUE>
```

### 7. Install the OpenTelemetry Operator

####  1. Deploy the cert-manager
```
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.6.1/cert-manager.yaml
```

Wait for pod webhook started
```
kubectl wait pod -l app.kubernetes.io/component=webhook -n cert-manager --for=condition=Ready --timeout=2m
```

#### 2. Deploy the opentelemetry operator
```
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml
```

### 7. Configure the OpenTelemetry Collector


#### Udpate the openTelemetry manifest file
```
CLUSTERID=$(kubectl get namespace kube-system -o jsonpath='{.metadata.uid}')
CLUSTERNAME="$NAME"
sed -i "s,CLUSTER_ID_TO_REPLACE,$CLUSTERID," kubernetes-manifests/openTelemetry-sidecar.yaml
sed -i "s,CLUSTER_NAME_TO_REPLACE,$CLUSTERNAME," kubernetes-manifests/openTelemetry-sidecar.yaml
sed -i "s,DT_TOKEN,$DATA_INGEST_TOKEN," kubernetes-manifests/openTelemetry-manifest.yaml
sed -i "s,DT_TENANT_URL,$DT_TENANT_URL," kubernetes-manifests/openTelemetry-manifest.yaml
```
#### Deploy the OpenTelemetry Collector
```
kubectl create ns otel
kubectl apply -f kubernetes-manifests/rbac.yaml -n otel
kubectl apply -f kubernetes-manifests/openTelemetry-manifest.yaml -n otel
```
### 8. Deploy the otel-demo
```
VERSION=v0.5.0-alpha
kubectl create ns otel-demo
sed -i "s,VERSION_TO_REPLACE,$VERSION," kubernetes-manifests/K8sdemo.yaml
kubectl apply -f kubernetes-manifests/openTelemetry-sidecar.yaml -n otel-demo
kubectl apply -f kubernetes-manifests/K8sdemo.yaml -n otel-demo
```

### 9. Deploy the Kepler
```
kubectl apply -f prometheus_exporter/deploy.yaml
kubectl apply -f prometheus_exporter/service_monitor.yaml
kubectl apply -f prometheus_exporter/kepler_dashboard.yaml
```

### 10. Update the Collector pipeline to send the data to Dynatrace
```
kubectl apply -f prometheus_exporter/deploy.yaml
kubectl apply -f prometheus_exporter/service_monitor.yaml
kubectl apply -f prometheus_exporter/kepler_dashboard.yaml
```

