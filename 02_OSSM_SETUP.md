
## Installing OpenShift Service Mesh 3 components (Operator/Istio/Gateway)

Installing OSSM3:
- [documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.1/html/installing/index)
- [operator installation](https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.1/html/installing/ossm-installing-service-mesh#ossm-about-deploying-istio-using-service-mesh-operator_ossm-installing-openshift-service-mesh)


To deploy Istio, you must create two resources: `Istio` and `IstioCNI`.

- The `Istio` resource deploys and configures the Istio Control Plane. 
- The `IstioCNI` resource deploys and configures the Istio Container Network Interface (CNI) plugin. 

You should create these resources in separate projects; therefore, you must create two projects as part of the Istio deployment process.

1. Create namespaces for Istio and IstioCNI

```bash
oc apply -f istio/namespaces.yaml
```

2. Create Istio control plane

Create the Istio resource that will contain the YAML configuration file for your Istio deployment. The Red Hat OpenShift Service Mesh Operator uses information in the YAML file to create an instance of the Istio control plane.

```bash
oc apply -f istio/istio.yaml  
oc wait --for condition=Ready istio/default --timeout 60s  -n istio-system   
```

This Istio also includes optional discovery selectors (currently commented out):
```yaml
spec:
  namespace: istio-system
  values:
    meshConfig:
      discoverySelectors:
        - matchLabels:
            istio-discovery: enabled
```

With discovery selectors, the mesh administrator can control which namespaces the control plane can access. By using a Kubernetes label selector, the administrator sets the criteria for the namespaces visible to the control plane, excluding any namespaces that do not match the specified criteria.

3. Create Istio Telemetry

Telemetry defines how telemetry (metrics, logs and traces) is generated for workloads within a mesh.
[Documentation](https://istio.io/latest/docs/reference/config/telemetry/)

```bash
oc apply -f istio/telemetry.yaml  
```

4. Create IstioCNI

Create an Istio Container Network Interface (CNI) resource, which contains the configuration file for the Istio CNI plugin. The Service Mesh Operator uses the configuration specified by this resource to deploy the CNI pod.


```bash
oc apply -f istio/istio-cni.yaml  
oc wait --for condition=Ready istiocni/default --timeout 60s -n istio-cni
```

## Ensure OpenTelemetry is picked up by the mesh

```bash
oc -n istio-system logs deploy/istiod | grep -i opentelemetry
```

Sample Output
```bash
opentelemetry:
    service: otel-collector.opentelemetrycollector.svc.cluster.local
2025-09-05T17:37:49.440154Z     info    model   Full push, new service opentelemetrycollector/otel-collector.opentelemetrycollector.svc.cluster.local
2025-09-05T17:37:49.440173Z     info    model   Full push, new service opentelemetrycollector/otel-collector-headless.opentelemetrycollector.svc.cluster.local
2025-09-05T17:37:49.440188Z     info    model   Full push, new service opentelemetrycollector/otel-collector-monitoring.opentelemetrycollector.svc.cluster.local
```

istiod log lines mean Istio has picked up your OpenTelemetry extension provider and discovered the Collector services

```bash
oc -n opentelemetrycollector get svc otel-collector -o wide
```

Sample Output
```bash
NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE   SELECTOR
otel-collector   ClusterIP   172.30.45.72   <none>        4317/TCP   19h   app.kubernetes.io/component=opentelemetry-collector,app.kubernetes.io/instance=opentelemetrycollector.otel,app.kubernetes.io/managed-by=opentelemetry-operator,app.kubernetes.io/part-of=opentelemetry
```

## Create ingress-gateway using gateway injection implementation and expose via Route

```bash
oc apply -f gateway-injection/namespace.yaml 
oc apply -f gateway-injection/gateway-deployment.yaml -n prod-gateway   
oc apply -f gateway-injection/route.yaml -n prod-gateway 
oc wait --for condition=Available deployment/istio-ingressgateway --timeout 60s -n prod-gateway  
```

## Enable workload monitoring in OpenShift and istio components

```bash
oc apply -f observability/servicemonitor.yaml -n istio-system  
oc apply -f observability/podmonitor.yaml -n istio-system    
oc apply -f observability/podmonitor.yaml -n prod-gateway 
```

## Install Kiali and OSSM Console plugin

```bash
oc apply -f kiali/cluster-role-binding.yaml -n istio-system
cat kiali/kiali.yaml | JAEGERROUTE="${TRACING_INGRESS_ROUTE}" envsubst | oc -n istio-system apply -f - 
oc wait --for condition=Successful kiali/kiali --timeout 150s -n istio-system 
oc annotate route kiali haproxy.router.openshift.io/timeout=60s -n istio-system 
oc apply -f kiali/ossm-console.yaml -n istio-system 
```

*Note This injects the Jaeger UI into the Kiali console* 
*Jaeger UI will be replaced with the Cluster Observability Operator (TODO)*


---
[Back to main README](/README.md) /
[Sample App (bookinfo)](/03_SAMPLE_APPS.md)

