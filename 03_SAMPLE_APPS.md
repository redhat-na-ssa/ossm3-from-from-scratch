# Sample applications to test OSSM3

Installing the `bookinfo` example application consists of two main tasks: deploying the application and creating a gateway so the application is accessible outside the cluster.

You can use the`bookinfo` application to explore service mesh features. Using the `bookinfo` application, you can easily confirm that requests from a web browser pass through the mesh and reach the application and test tracing

## Install Sample Application (bookinfo) (GW Injection)

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

## Bookinfo Traffic Generator (optional)

```bash
export INGRESSHOST=$(oc get route istio-ingressgateway -n prod-gateway -o=jsonpath='{.spec.host}')
cat ./bookinfo/bookinfo-traffic-gen/traffic-generator-configmap.yaml | ROUTE="https://${INGRESSHOST}/productpage" envsubst | oc -n bookinfo apply -f - 
oc apply -f ./bookinfo/bookinfo-traffic-gen/traffic-generator.yaml -n bookinfo
```

*Note: The latest source for the bookinfo app deployment can be found at* 
https://raw.githubusercontent.com/openshift-service-mesh/istio/release-1.24/samples/bookinfo/platform/kube/bookinfo.yaml

---

[Back to main README](/README.md) /
[Gateway API Ingress)](/04_GATEWAY_API)