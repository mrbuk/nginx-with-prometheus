# nginx with prometheus metrics


We will look into
 * how to expose nginx metrics via promtheus
 * deploy nginx + exporter side car to k8s
 * nginx + prometheus exporter on k8s
 * collect metrics on GKE using Workload Metrics
 * testing application scaling

## quickstart

```
cd k8s

kubectl apply -f nginx-with-prometheus.yaml

# ensure workload metrics are enabled and the
# service account has 'role/monitoring.metricWriter'
# before running the following command
kubectl apply -f pod-monitor.yaml

# install the custom metric adapter to make
# Cloud Monitoring metrics with K8s available
kubectl create clusterrolebinding cluster-admin-binding \
    --clusterrole cluster-admin --user "$(gcloud config get-value account)"
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/k8s-stackdriver/master/custom-metrics-stackdriver-adapter/deploy/production/adapter_new_resource_model.yaml

# create HPA based on waiting connections
kubectl -f hpa.yaml

# install 'benchmarking' tool
go get github.com/tsliwowicz/go-wrk

# run benchmark and observe in parallel with 
# `kubectl get pods -w` how pods are scaled up
SERVER_IP=$(kubectl get svc nginx-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
go-wrk -d 1000 -c 10 http://$SERVER_IP
```

## how to expose nginx metrics via prometheus

The [nginx Docker image](https://hub.docker.com/_/nginx) does not expose
metrics per default. To expose metrics metrics we extend the `nginx` docker
image and add `stub_status.conf` configuration. This will expose metrics on
port `:8080/nginx_status`

```
server { 
 listen 8080;
  location /nginx_status {
    stub_status;
    allow 127.0.0.1;
    deny all;
  }
}
```

`nginx` metrics are not compatible with [Prometheus Notion for Metrics](https://github.com/nginxinc/nginx-prometheus-exporter) out of the box. To expose the `nginx` metrics in a Prometheus Notion the [nginx-promtheus-exporter](https://github.com/nginxinc/nginx-prometheus-exporter) can be used.

## deploy nginx + exporter side car to k8s

The `k8s` directory contains a [deployment file](k8s/nginx-with-prometheus.yml) for Kubernetes that deploys the exporter as side-car to the `nginx` container. That way it should be possible to scrape `nginx` metrics by any prometheus comaptbile software.

When deployed the metrics can be fetched via `http://IP:9113/metrics`:

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

## collect metrics on GKE using Workload Metrics

When using [Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine) these metrics can be scraped using the [Workload Metrics](https://cloud.google.com/blog/products/operations/managed-metric-collection-for-google-kubernetes-engine) feature that is currently (Dec 2021) in Preview.

`k8s` contains a [pod-monitor.yaml](k8s/pod-monitor.yaml) file that will scrape metrics from `nginx` and make them available in [Cloud Monitoring][https://console.cloud.google.com/monitoring/metrics-explorer] if `Workload Metrics` are enabled. You should be able to search for `nginx_connections_accepted` in the Metrics Explorer.

## scale based on metrics
`k8s` contains a [hpa.yaml](k8s/hpa.yaml) that scales the cluster based on the `nginx_connections_waiting` metric. Before you can can apply the Horizontal Autoscaler config the custom metrics adapter needs to be installed as described in the [documentation](https://cloud.google.com/kubernetes-engine/docs/tutorials/autoscaling-metrics#step1):

```
kubectl create clusterrolebinding cluster-admin-binding \
    --clusterrole cluster-admin --user "$(gcloud config get-value account)"

kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/k8s-stackdriver/master/custom-metrics-stackdriver-adapter/deploy/production/adapter_new_resource_model.yaml

```

## testing application scaling

Running the following two commands should lead to a increased number of `nginx` pods (up to 10)
```
go get github.com/tsliwowicz/go-wrk
go-wrk -d 1000 -c 10 http://$SERVER_IP
```

