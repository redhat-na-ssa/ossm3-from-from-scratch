

# Observability Integration

This document follows the official Red Hat documentation
[Observability and Service Mesh](https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.1/html/observability/index)
[OCP documentation: Configuring user workload monitoring](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/monitoring/configuring-user-workload-monitoring#configuring-performance-and-scalability-uwm)
[Cluster Observability Operator](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/cluster_observability_operator/index)


Observability is provided by OpenShift monitoring (CMO + UWM) and OpenShift distributed tracing operators.

We need configure user workload monitoring to collect Istio metrics and deploy tracing/Kiali separately.

## Metrics and Service Mesh (as cluster administrator)

Monitoring stack components are deployed by default in every OpenShift Container Platform installation and are managed by the Cluster Monitoring Operator (CMO). These components include Prometheus, Alertmanager, Thanos Querier, and others. The CMO also deploys the Telemeter Client, which sends a subset of data from platform Prometheus instances to Red Hat to facilitate Remote Health Monitoring for clusters.

The Cluster Monitoring Operator (CMO) is installed by default in OCP as part of the cluster-monitoring stack.

1. Configuring user workload monitoring

User workload monitoring is a feature that rquires an explicit opt-in. It gives application namespaces their own Prometheus + Thanos Ruler stack.

This involves creating a new ConfigMap `cluster-monitoring-config` in the `openshift-monitorinng` namespace with `enableUserWorkload: true`.

```bash
oc apply -f observability/configmap.yaml    
```

Output
```bash
configmap/cluster-monitoring-config created
```

Verify that the `prometheus-operator`, `prometheus-user-workload`, and `thanos-ruler-user-workload` pods are running in the openshift-user-workload-monitoring project.

```bash
oc get pods -n openshift-user-workload-monitoring   
```

Output:
```bash
NAME                                   READY   STATUS    RESTARTS   AGE
prometheus-operator-854db99d74-v2fww   2/2     Running   0          117s
prometheus-user-workload-0             6/6     Running   0          116s
thanos-ruler-user-workload-0           4/4     Running   0          115s
```

2. Create a `ServiceMonitor` to monitor the Istio control plane

*Note assumes you have create istio namespaces already*



```bash 
oc apply -f observability/servicemonitor.yaml -n istio-system
```

Output
```bash
servicemonitor.monitoring.coreos.com/istiod-monitor created
```

3. Create a PodMonitor to collect metrics from the Istio proxies 

```bash
oc apply -f observability/podmonitor.yaml -n istio-system 
oc apply -f observability/podmonitor.yaml -n prod-gateway 
oc apply -f observability/podmonitor.yaml -n bookinfo  
```

Output
```bash
podmonitor.monitoring.coreos.com/istio-proxies-monitor created
```

*Note: You will need to deploy a PodMonitor in each application and gateway namespace managed by the service mesh*

4. Ensure observability for service mesh is working correctly from the OpenShift Console:
    - Go to Observe->Metrics
    - Run the query `istio_requests_total`

![Metrics Query Example](/img/image01.png)

## Configuring Red Hat OpenShift distributed tracing platform with Service Mesh 

Integrating Red Hat OpenShift distributed tracing platform with Red Hat OpenShift Service Mesh is made of up two parts: Red Hat OpenShift distributed tracing platform (Tempo) and Red Hat OpenShift distributed tracing data collection.

### Tempo installation and configuration
[Source](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/distributed_tracing/distr-tracing-architecture)

1. Install the Tempo Operator

```bash
oc apply -f operators/tempo-product.yaml 
```

Output
```bash
namespace/openshift-tempo-operator created
operatorgroup.operators.coreos.com/openshift-tempo-operator created
subscription.operators.coreos.com/tempo-product created
```
Check to make sure the `tempo-operator` `PHASE` is in the `Succeeded` state

```bash
oc get csv -n openshift-tempo-operator
```

Output:
```bash
NAME                          DISPLAY                            VERSION    REPLACES                      PHASE
...
tempo-operator.v0.16.0-2      Tempo Operator                     0.16.0-2   tempo-operator.v0.16.0-1      Succeeded
```

2. Set up a supported object store and creating a secret for the object store credentials.

For this section, we will install and configure MinIO for object storage, but you can use AWS S3 instead.
[Amazon S3 documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/create-bucket-overview.html)
[MinIO installation guide](https://operator.min.io/#deploy-the-minio-operator-and-create-a-tenant)

- Install MinIO



```bash
oc apply -f minio/namespace.yaml 
oc apply -f minio/minio-creds.yaml -n minio
oc apply -f minio/minio.yaml -n minio 
```

Output:
```bash
namespace/minio created
secret/minio-creds created
statefulset.apps/minio created
service/minio created
route.route.openshift.io/minio-api created
route.route.openshift.io/minio-console created
```

*Note: This is done in two steps so we can reuse the `secret/minio-creds` in the next step*

To login to the MinIO console
```bash
export MINIO_CONSOLE=$(oc -n minio get route minio-console -o jsonpath='{.spec.host}')
echo https://$MINIO_CONSOLE 
```

Default username and password has been set in the secret `secret/minio-creds` as:
```yaml
  accesskey: minioadmin
  secretkey: minioadmin123
``` 

Feel free to modify these.

- Create bucket for tempo data using a job
```bash
oc apply -f minio/minio-create-tempo-bucket.yaml 
```

Once the job is complete, you can check that the bucket was created by refreshing the MinIO web UI.
![tempo-data bucket](/img/image02.png)

3. Create Tracing System (Tempo-Stack)

- Prepare namespace and secret for connecting to object storage

```bash
oc apply -f tracing/namespace.yaml  
oc apply -f minio/minio-creds.yaml -n tracing-system
```

- Deploy TempoStack
```bash
oc apply -f tracing/tempostack.yaml -n tracing-system
```

- Check the status of the tempo-stack components to make sure they are all up and running

```bash
oc get all -n tracing-system 
```
Output
```bash
NAME                                               READY   STATUS    RESTARTS   AGE
pod/tempo-sample-compactor-547f94c8c6-mmwcr        1/1     Running   0          100s
pod/tempo-sample-distributor-6c6d56956-5wfdv       1/1     Running   0          100s
pod/tempo-sample-ingester-0                        1/1     Running   0          100s
pod/tempo-sample-querier-5cdcc6b467-cqt4l          1/1     Running   0          100s
pod/tempo-sample-query-frontend-79944854f5-b4jdl   3/3     Running   0          100s

NAME                                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                     AGE
service/tempo-sample-compactor                  ClusterIP   172.30.66.154    <none>        7946/TCP,3200/TCP                                                           100s
service/tempo-sample-distributor                ClusterIP   172.30.186.240   <none>        4318/TCP,4317/TCP,3200/TCP,14268/TCP,6831/UDP,6832/UDP,14250/TCP,9411/TCP   100s
service/tempo-sample-gossip-ring                ClusterIP   None             <none>        7946/TCP                                                                    100s
service/tempo-sample-ingester                   ClusterIP   172.30.117.217   <none>        3200/TCP,9095/TCP                                                           100s
service/tempo-sample-querier                    ClusterIP   172.30.145.243   <none>        7946/TCP,3200/TCP,9095/TCP                                                  100s
service/tempo-sample-query-frontend             ClusterIP   172.30.192.106   <none>        3200/TCP,9095/TCP,16685/TCP,16686/TCP,16687/TCP                             100s
service/tempo-sample-query-frontend-discovery   ClusterIP   None             <none>        3200/TCP,9095/TCP,9096/TCP,16685/TCP,16686/TCP,16687/TCP                    100s

NAME                                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/tempo-sample-compactor        1/1     1            1           100s
deployment.apps/tempo-sample-distributor      1/1     1            1           100s
deployment.apps/tempo-sample-querier          1/1     1            1           100s
deployment.apps/tempo-sample-query-frontend   1/1     1            1           100s

NAME                                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/tempo-sample-compactor-547f94c8c6        1         1         1       100s
replicaset.apps/tempo-sample-distributor-6c6d56956       1         1         1       100s
replicaset.apps/tempo-sample-querier-5cdcc6b467          1         1         1       100s
replicaset.apps/tempo-sample-query-frontend-79944854f5   1         1         1       100s

NAME                                     READY   AGE
statefulset.apps/tempo-sample-ingester   1/1     100s
```

- To get to the url of the tracing UI

```bash
export TRACING_ROUTE=$(oc get route -n tracing-system tempo-tempo-stack-gateway -o jsonpath='{.spec.host}')
export TRACING_UI=https://$TRACING_ROUTE/dev 
echo $TRACING_UI 
```

## Cluster Observability Operator and the Cluster Observability Operator distributed tracing UI plugin

[Documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/cluster_observability_operator/cluster-observability-operator-overview)

The `Cluster Observability Operator (COO)` is an optional component of the OpenShift Container Platform 
designed for creating and managing highly customizable monitoring stacks. It enables cluster administrators 
to automate configuration and management of monitoring needs extensively, offering a more tailored and 
detailed view of each namespace compared to the default OpenShift Container Platform monitoring system.  

The COO components function independently of the default in-cluster monitoring stack, which is deployed 
and managed by the Cluster Monitoring Operator (CMO). Monitoring stacks deployed by the two Operators do 
not conflict. You can use a COO monitoring stack in addition to the default platform monitoring components 
deployed by the CMO.

The distributed tracing UI plugin adds tracing-related features to the OpenShift Container Platform web 
console at Observe -> Traces. You can follow requests through the front end and into the backend of 
microservices, helping you identify code errors and performance bottlenecks in distributed systems.

### Installing The `Cluster Observability Operator (COO)`

1. Install the COO operator

```bash
oc apply -f operators/cluster-observability-operator.yaml  
```

2. Install the distributed tracing UI plugin

```bash
oc apply -f plugins/coo-ui-plugin.yaml  
```

## Red Hat OpenShift distributed tracing data collection

[Documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/red_hat_build_of_opentelemetry/index)

You can use the Red Hat build of OpenTelemetry in combination with the Red Hat OpenShift Distributed Tracing Platform.

Red Hat build of OpenTelemetry is based on the open source OpenTelemetry project, which aims to provide unified, 
standardized, and vendor-neutral telemetry data collection for cloud-native software. Red Hat build of OpenTelemetry 
product provides support for deploying and managing the OpenTelemetry Collector and simplifying the workload instrumentation.

The OpenTelemetry Collector can receive, process, and forward telemetry data in multiple formats, making it the 
ideal component for telemetry processing and interoperability between telemetry systems. The Collector provides 
a unified solution for collecting and processing metrics, traces, and logs.

1. Install the Red Hat build of OpenTelemetry Operator

```bash
oc apply -f operators/opentelemetry-product.yaml    
```

2. Create an `opentelemetrycollector`

```bash
oc apply -f open-telemetry/opentelemetrycollector.yaml
```

TODO test to see if tracing metrics can be seen
---
[Back to main README](/README.md)