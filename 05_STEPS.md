1. Install Operators

```bash
oc apply -f operators  

until oc get pods -n openshift-operators | grep servicemesh-operator3 | grep Running; do echo "Waiting for servicemesh-operator3 to be running."; sleep 10;done
until oc get pods -n openshift-operators | grep kiali-operator | grep Running; do echo "Waiting for kiali-operator to be running."; sleep 10;done
until oc get pods -n openshift-opentelemetry-operator | grep opentelemetry-operator | grep Running; do echo "Waiting for opentelemetry-operator to be running."; sleep 10;done
until oc get pods -n openshift-tempo-operator | grep tempo-operator | grep Running; do echo "Waiting for tempo-operator to be running."; sleep 10;done
until oc get pods -n openshift-cluster-observability-operator | grep observability-operator | grep Running; do echo "Waiting for observability-operator to be running."; sleep 10;done
```

2. install Minio for Tempo

```bash
oc apply -f minio/namespace.yaml 
oc apply -f minio/minio-creds.yaml -n minio
oc apply -f minio/minio.yaml -n minio 
```

MinIO console
```bash
export MINIO_CONSOLE=$(oc -n minio get route minio-console -o jsonpath='{.spec.host}')
echo https://$MINIO_CONSOLE 
```

3. Create bucket for tempo-data

```bash
oc apply -f minio/minio-create-tempo-bucket.yaml 
```

4. Enable user workload monitoring

```bash
oc apply -f observability/configmap.yaml 
```
5. enable cluster observability operator UIPlugin for tracing UI

```bash
oc create -k tempoStack-coo/observability-plugin
```

5. Install the TempoStack with Multi Tenancy enabled


```bash
oc create -k tempoStack-coo/tempoStack
```


7. Install Service Mesh resources

```bash
oc apply -f istio/namespaces.yaml
oc apply -f istio/istio.yaml  
oc wait --for condition=Ready istio/default --timeout 60s  -n istio-system
oc apply -f istio/telemetry.yaml  
oc apply -f istio/istio-cni.yaml  
oc wait --for condition=Ready istiocni/default --timeout 60s -n istio-cni
```

8. Create ingress-gateway using gateway injection implementation and expose via Route

```bash
oc apply -f gateway-injection/namespace.yaml 
oc apply -f gateway-injection/gateway-deployment.yaml -n prod-gateway   
oc apply -f gateway-injection/route.yaml -n prod-gateway 
oc wait --for condition=Available deployment/istio-ingressgateway --timeout 60s -n prod-gateway  
```

9. Enable workload monitoring in OpenShift and istio components

```bash
oc apply -f observability/configmap.yaml 
oc apply -f observability/servicemonitor.yaml -n istio-system  
oc apply -f observability/podmonitor.yaml -n istio-system    
oc apply -f observability/podmonitor.yaml -n prod-gateway 
```
10. Install Kiali

```bash
oc apply -f kiali/cluster-role-binding.yaml -n istio-system
cat kiali/kiali.yaml | JAEGERROUTE="${TRACING_INGRESS_ROUTE}" envsubst | oc -n istio-system apply -f - 
oc wait --for condition=Successful kiali/kiali --timeout 150s -n istio-system 
oc annotate route kiali haproxy.router.openshift.io/timeout=60s -n istio-system 
oc apply -f kiali/ossm-console.yaml -n istio-system 
```

11. Install Sample Application (bookinfo) (GW Injection)

```bash
oc apply -f bookinfo/namespace.yaml   
oc apply -f observability/podmonitor.yaml -n bookinfo # super important! 
oc apply -f bookinfo/bookinfo.yaml -n bookinfo 
oc apply -f gateway-injection/gateway-injection.yaml -n prod-gateway 
oc apply -f bookinfo/virtualservice.yaml -n bookinfo 
oc wait --for=condition=Ready pods --all -n bookinfo --timeout 60s
HOST=$(oc get route istio-ingressgateway -n prod-gateway -o jsonpath='{.spec.host}')
echo productpage URL: https://$HOST/productpage
```

12. Bookinfo Traffic Generator (optional)

```bash
export INGRESSHOST=$(oc get route istio-ingressgateway -n prod-gateway -o=jsonpath='{.spec.host}')
cat ./bookinfo/bookinfo-traffic-gen/traffic-generator-configmap.yaml | ROUTE="https://${INGRESSHOST}/productpage" envsubst | oc -n bookinfo apply -f - 
oc apply -f ./bookinfo/bookinfo-traffic-gen/traffic-generator.yaml -n bookinfo
```