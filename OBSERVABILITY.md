

# Observability Integration

This document follows the official Red Hat documentation
[Observability and Service Mesh](https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.1/html/observability/index)
[OCP documentation: Configuring user workload monitoring](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/monitoring/configuring-user-workload-monitoring#configuring-performance-and-scalability-uwm)

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
prometheus-user-workload-1             6/6     Running   0          116s
thanos-ruler-user-workload-0           4/4     Running   0          115s
thanos-ruler-user-workload-1           4/4     Running   0          115s
```

2. Create a `ServiceMonitor` to monitor the Istio control plane

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





