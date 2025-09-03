# ossm3-from-from-scratch
Following official RH documentation

Tested on
```bash
Client Version: 4.19.7
Kustomize Version: v5.5.0
Server Version: 4.18.22
Kubernetes Version: v1.31.11
```

## Installing OpenShift Service Mesh 3 components

Installing OSSM3:
- [documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.1/html/installing/index)
- [operator installation](https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.1/html/installing/ossm-installing-service-mesh#ossm-about-deploying-istio-using-service-mesh-operator_ossm-installing-openshift-service-mesh)

1. Install the OpenShift Service Mesh 3 Operator

```bash
oc apply -f operators/
```

2. Deploy Service Mesh resources

To deploy Istio, you must create two resources: `Istio` and `IstioCNI`.

- The `Istio` resource deploys and configures the Istio Control Plane. 
- The `IstioCNI` resource deploys and configures the Istio Container Network Interface (CNI) plugin. 

You should create these resources in separate projects; therefore, you must create two projects as part of the Istio deployment process.

3. Create namespaces for Istio and IstioCNI

```bash
oc apply -f istio/namespaces.yaml
```

Output:
```bash
namespace/istio-system created
namespace/istio-cni created
```

4. Create Istio control plane

Create the Istio resource that will contain the YAML configuration file for your Istio deployment. The Red Hat OpenShift Service Mesh Operator uses information in the YAML file to create an instance of the Istio control plane.

```bash
oc apply -f istio/istio.yaml     
```
Output:
```bash
istio.sailoperator.io/default created
```

This Istio also includes optional discovery selectors:
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

5. Create IstioCNI

Create an Istio Container Network Interface (CNI) resource, which contains the configuration file for the Istio CNI plugin. The Service Mesh Operator uses the configuration specified by this resource to deploy the CNI pod.


```bash
oc apply -f istio/istio-cni.yaml  
```

Output:
```bash
istiocni.sailoperator.io/default created
```

## Deploy test application (Bookinfo)

Installing the `bookinfo` example application consists of two main tasks: deploying the application and creating a gateway so the application is accessible outside the cluster.

You can use the`bookinfo` application to explore service mesh features. Using the `bookinfo` application, you can easily confirm that requests from a web browser pass through the mesh and reach the application.

1. Create the `bookinfo` namespace

```bash
oc apply -f bookinfo/namespace.yaml   
```

Output:
```bash
namespace/bookinfo created
```

2. Deploy the `bookinfo` application

```bash
oc apply -f bookinfo/bookinfo.yaml -n bookinfo 
```

Output:
```bash
service/details created
serviceaccount/bookinfo-details created
deployment.apps/details-v1 created
service/ratings created
serviceaccount/bookinfo-ratings created
deployment.apps/ratings-v1 created
service/reviews created
serviceaccount/bookinfo-reviews created
deployment.apps/reviews-v1 created
deployment.apps/reviews-v2 created
deployment.apps/reviews-v3 created
service/productpage created
serviceaccount/bookinfo-productpage created
deployment.apps/productpage-v1 created
```

*Note: The latest source for the bookinfo app deployment can be found at* 
https://raw.githubusercontent.com/openshift-service-mesh/istio/release-1.24/samples/bookinfo/platform/kube/bookinfo.yaml

3. Verify that the bookinfo pods are available

```bash
 oc get pods -n bookinfo 
```

Output:
```bash
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-7c799b8b4b-9zd2r       2/2     Running   0          116s
productpage-v1-77f599b985-gfxxn   2/2     Running   0          114s
ratings-v1-7fccfc8b8b-t7n2t       2/2     Running   0          116s
reviews-v1-746bcb79b-ddwbt        2/2     Running   0          115s
reviews-v2-5bfff8fb5d-v7hm5       2/2     Running   0          115s
reviews-v3-7879bc9c5b-fdgvp       2/2     Running   0          114s
```
When the `READY` columns displays `2/2`, the proxy sidecar was successfully injected. Confirm that Running appears in the `STATUS` column for each pod.

4. Test internal access to the bookinfo application

```bash
oc exec "$(oc get pod -l app=ratings -n bookinfo -o jsonpath='{.items[0].metadata.name}')" -c ratings -n bookinfo -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
```

Output:
```html
<title>Simple Bookstore App</title>
```

## Exposing the `bookinfo` application with a gateway

The Red Hat OpenShift Service Mesh Operator does not deploy gateways. Gateways are not part of the control plane. As a security best-practice, Ingress and Egress gateways should be deployed in a different namespace than the namespace that contains the control plane.

You can deploy gateways using either the Gateway API or the gateway injection method.

### Istio gateway injection method

1. create namespace for the gateway called `prod-gateway`

```bash
oc apply -f gateway-injection/namespace.yaml     
```

Output
```bash
namespace/prod-gateway created
```

2. Create the `istio-ingressgateway` deployment and service

```bash
oc apply -f gateway-injection/gateway-deployment.yaml -n prod-gateway 
```

Output
```bash
service/istio-ingressgateway created
deployment.apps/istio-ingressgateway created
role.rbac.authorization.k8s.io/istio-ingressgateway-sds created
rolebinding.rbac.authorization.k8s.io/istio-ingressgateway-sds created
```


3. Configure the `bookinfo` application to use the new gateway

```bash
oc apply -f bookinfo/gateway-injection.yaml -n prod-gateway 
```

Output:
```bash
gateway.networking.istio.io/bookinfo-gateway created
```

*Note: Here we put the Gateway injection in the gateway namespace (`prod-gateway`), assuming 
the platform team owns ingress, and the application team publishes routes. We 
could have created this in the `bookinfo` namespace if app teams are fairly self-sufficient and you want them to own their own ingress config.*


4. Use a route to expose the gateway external to the cluster

```bash
oc apply -f gateway-injection/route.yaml -n prod-gateway 
```

Output:
```bash
route.route.openshift.io/istio-ingressgateway created
```

5. Associate the `bookinfo` app's service with the `bookinfo-gateway` using a VirtualService definition

```bash
oc apply -f bookinfo/virtualservice.yaml -n bookinfo  
```

Output
```bash
virtualservice.networking.istio.io/bookinfo created
```

```bash
oc get virtualservice -n bookinfo  
```
```bash                             
NAME       GATEWAYS                            HOSTS   AGE
bookinfo   ["prod-gateway/bookinfo-gateway"]   ["*"]   77s
```

6. Access the bookinfo app through the gateway

```bash
HOST=$(oc get route istio-ingressgateway -n prod-gateway -o jsonpath='{.spec.host}')
echo productpage URL: https://$HOST/productpage
```

TODO: 
- implement with Gatway API
- deploy travel agency app
- observability
- distributed tracing
- kiali
- cert management