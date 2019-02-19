# Deploy Monitoring on K8S
This guide will help you deploy monitoring tools on Kubernetes that was deployed by the [kops](https://github.com/kubernetes/kops). In this I will skip `How to deploy Kubernetes on AWS by the [kops](https://github.com/kubernetes/kops)`. For deploy Kubernetes by the kops you can refer guide line [here](https://github.com/kubernetes/kops).

I have references the docs below to created this docs :
- https://github.com/pires/kubernetes-elasticsearch-cluster
- https://github.com/coreos/prometheus-operator/tree/master/contrib/kube-prometheus
- https://blog.ptrk.io/how-to-deploy-an-efk-stack-to-kubernetes/

## Pre-requisites
- Kubernetes 1.11.x.
- kubectl configured to access the Kubernetes API.

This include 2 parts:

## Log
We use Elasticsearch, Fluentd and Kibana (EFK).

### Overview
The Elasticsearch cluster on Kubernetes include:

- Master nodes - intended for clustering management only, no data, no HTTP API.
- Data nodes - intended for client usage and data.
- Ingest nodes - intended for document pre-processing during ingestion.


### Deploy
We deploy EFK in namespace `logging`

Create namespace: `kubectl create ns logging`


Deploy Elasticsearch
- Minimum 2 master
```
kubectl create -f logging/es-discovery-svc.yaml
kubectl create -f logging/es-svc.yaml
kubectl create -f logging/es-master.yaml

kubectl create -f logging/es-ingest-svc.yaml
kubectl create -f logging/es-ingest.yaml

kubectl create -f logging/es-data.yaml
```

Check:
```
kubectl get svc,deployment,pods -n=logging
```

```
curl http://ip_es_service:9200/_cluster/health?pretty
```

- In advantage, If you wants to ensure that no more than n Elasticsearch nodes will be unavailable at a time, one can optionally (change and) apply the following manifests:
```
kubectl create -f logging/advance/es-master-pdb.yaml
kubectl create -f logging/advance/es-data-pdb.yaml
```

Note: This is an advanced subject and one should only put it in practice if one understands clearly what it means both in the Kubernetes and Elasticsearch contexts. For more information, please consult [Pod Disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/).

- Additionally, one can run a CronJob that will periodically run Curator to clean up indices (or do other actions on the Elasticsearch cluster).
```
kubectl apply -f logging/crond/es-curator-config.yaml
kubectl apply -f logging/crond/es-curator.yaml
```

- Refer [Here](logging/stateful/README.md) If you want to deploy elasticsearch data and master pods as a StatefulSet, using storage provisioned using a StorageClass.

Deploy Kibana
```
kubectl create -f logging/kibana-dep.yaml
kubectl create -f logging/kibana-ingres.yaml
```

Check:
- Get ELB address (I skip how to deploy [Ingress](https://github.com/kubernetes/ingress-nginx) on K8s)
```
kubectl get ing -n=logging
```

- Get IP address of ELB
```
host aws_ingress_elb.us-east-2.elb.amazonaws.com
```

- Mapping domain `kibana.staging.ts` to ELB IP address by add to `/etc/hosts` or if you want to use public domain, you can create a Alias point to ELB domain in your manage domain.

Deploy Fluentd and Fluentbit
```
kubectl create -f logging/fluentd.yaml
kubectl create -f logging/fluent-bit-configmap.yaml
kubectl create -f logging/fluentbit.yaml
```

Check: 

Open browse and access `kibana.staging.ts` you will see the log of Node, Container, ...

## Metrics
We use Prometheus and Grafana. This is base on the [Prometheus Operator](https://github.com/coreos/prometheus-operator) project. 

In this docs I have created manifest files deployment in metrics folder. If you want to manual to create manifest files you can refer [Here](https://github.com/coreos/prometheus-operator/tree/master/contrib/kube-prometheus).

Deploy

We deploy Metrics in namespace `monitoring`

Create namespace: `kubectl create ns monitoring`

```
kubectl create -f metrics/
```

Delete
```
kubectl delete -f metrics/
```

# Questions
For questions and support please create a issue in this repo.