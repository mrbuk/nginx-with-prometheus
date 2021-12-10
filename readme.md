**Please be aware: this is not an official code sample or project. It is provided for illustrative purposes only.**

# nginx with prometheus metrics

[Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine) introduced a feature in Preview _(as of Dec 2021)_ called [Workload Metrics](https://cloud.google.com/blog/products/operations/managed-metric-collection-for-google-kubernetes-engine). Workload Metrics integrates custom metrics, exposed by workload deployed in the GKE cluster, into Cloud Monitoring. Once the metrics are in Cloud Monitoring they can be used to create dashboards or for autoscaling workloads (e.g. using _Horizontal Pod Autoscaler (HPA)_).

With `nginx` as a sample workload we will look into:
 1. [How to expose nginx metrics via prometheus](#how-to-expose-nginx-metrics-via-prometheus)
 1. [Deploy nginx with a prometheus exporter sidecar to Kubernetes](#deploy-nginx-with-a-prometheus-exporter-sidecar-to-kubernetes)
 1. [Publish nginx metrics to Cloud Monitoring (Workload Metrics)](#publish-nginx-metrics-to-cloud-monitoring-workload-metrics)
 1. [Scale based deployment HPA and Cloud Monitoring (Workload Metrics)](#scale-based-deployment-hpa-and-cloud-monitoring-workload-metrics)

## Quickstart without explanation

If you prefer to skip the explanation you can run the following commands:

```bash
cd k8s

kubectl apply -f nginx-with-prometheus.yaml

# ensure workload metrics are enabled 
kubectl get crd podmonitors.monitoring.gke.io

# check if gke service account has 'role/monitoring.metricWriter' 
gcloud projects get-iam-policy $PROJECT \
 --flatten="bindings[].members" \
 --format="table(bindings.role)" \
 --filter="bindings.members:$GKE_SA"

# expose nginx metrics via workload metrics
kubectl apply -f pod-monitor.yaml

# install the custom metric adapter to make Cloud Monitoring metrics with K8s available
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole cluster-admin \
  --user "$(gcloud config get-value account)"

# check the redirect: curl -s --head https://git.io/JDYP1 | grep 'Location'
kubectl apply -f https://git.io/JDYP1

# create HPA based on waiting connections
kubectl -f hpa.yaml

# install 'benchmarking' tool
go get github.com/tsliwowicz/go-wrk

# run benchmark and observe in parallel with `kubectl get pods -w` how pods are scaled up
SERVER_IP=$(kubectl get svc nginx-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
# run for 10 minutes with 10 connections
go-wrk -d 600 -c 10 http://$SERVER_IP
```

## How to expose nginx metrics via prometheus

The [nginx Docker image](https://hub.docker.com/_/nginx) does not expose metrics in the default configuration. To expose metrics we need to add a `stub_status.conf` configuration file to the image: 

```Nginx
server { 
 listen 8080;
  location /nginx_status {
    stub_status;
    allow 127.0.0.1;
    deny all;
  }
}
```

With that configuration file in place metrics will be exposed at `http://CONTAINER_IP:8080/nginx_status`

[Dockerfile](docker/Dockerfile) shows how to add that configuration to the `nginx` container image. To build your own version of the container image run:

```bash
cd docker
docker build -t whatever-you-like/nginx:1.21 .
docker push whatever-you-like/nginx:1.21 
```

For the time being you can use `mrbuk/nginx:1:21` for this but be aware that it might be removed in future.

## Deploy nginx with a prometheus exporter sidecar to Kubernetes

Now that we have `nginx` metrics we need to make them compatible with the [Prometheus Notion for Metrics](https://github.com/nginxinc/nginx-prometheus-exporter) as this is what Workload Metrics expects. To do so we use the [nginx-promtheus-exporter](https://github.com/nginxinc/nginx-prometheus-exporter) OSS project.

Instead of deploying the prometheus exporter separately we bundle it as a sidecar to the `nginx` container in the same pod. [Deployment file](k8s/nginx-with-prometheus.yml) shows how to do that. 

Once deployed and running metrics should be available at `http://IP:9113/metrics` and look like:

```
# comments with type/help removed
nginx_connections_accepted 4114
nginx_connections_active 1
nginx_connections_handled 4114
nginx_connections_reading 0
nginx_connections_waiting 0
nginx_connections_writing 1
nginx_http_requests_total 6928
nginx_up 1
nginxexporter_build_info{commit="5f88afbd906baae02edfbab4f5715e06d88538a0",date="2021-03-22T20:16:09Z",version="0.9.0"} 1
```

## Publish nginx metrics to Cloud Monitoring (Workload Metrics)

Workload Metrics deploys a CRD called `k get crd podmonitors.monitoring.gke.io` into defined a `PodMonitor` `k8s` contains a [pod-monitor.yaml](k8s/pod-monitor.yaml) file that will scrape metrics from `nginx` and make them available in [Cloud Monitoring](https://console.cloud.google.com/monitoring/metrics-explorer) if `Workload Metrics` is enabled. 

Once you deployed it you should be able to search for `nginx_connections_accepted` in the Metrics Explorer.

```bash
# ensure workload metrics are enabled 
kubectl get crd podmonitors.monitoring.gke.io

# check if gke service account has 'role/monitoring.metricWriter' 
gcloud projects get-iam-policy $PROJECT \
 --flatten="bindings[].members" \
 --format="table(bindings.role)" \
 --filter="bindings.members:$GKE_SA"

# expose nginx metrics via workload metrics
kubectl apply -f pod-monitor.yaml
```

## Scale based deployment HPA and Cloud Monitoring (Workload Metrics)

To apply the Horizontal Autoscaler config the [custom metrics adapter](https://cloud.google.com/kubernetes-engine/docs/tutorials/autoscaling-metrics#step1) needs to be installed. With the custom metrics adapter deployed we can use [a HPA definition](k8s/hpa.yaml) to scale `nginx` based on the number of waiting connections (`nginx_connections_waiting`).

```bash
# install the custom metric adapter to make Cloud Monitoring metrics with K8s available
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole cluster-admin \
  --user "$(gcloud config get-value account)"

# check the redirect: curl -s --head https://git.io/JDYP1 | grep 'Location'
kubectl apply -f https://git.io/JDYP1

# create HPA based on waiting connections
kubectl -f hpa.yaml
```

To test application scaling we can run the following two commands (assuming `go` is installed):

```bash
# skip installation if you have the binary already or use wrk instead
go get github.com/tsliwowicz/go-wrk

# run benchmark and observe in parallel with `kubectl get pods -w` how pods are scaled up
SERVER_IP=$(kubectl get svc nginx-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
# run for 10 minutes with 10 connections
go-wrk -d 600 -c 10 http://$SERVER_IP
```

while the `go-wrk` command is running you should after some time start to see an increased number of `nginx` pods (up to 10). If you stop `go-wrk` you will see the pods being scaled down back to 1.

