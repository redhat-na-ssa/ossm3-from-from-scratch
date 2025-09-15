# ossm3-from-from-scratch
Following official RH documentation
---
Tested on
```bash
Client Version: 4.19.7
Kustomize Version: v5.5.0
Server Version: 4.18.23
Kubernetes Version: v1.31.11
```
## Quick Setup 

You can manually install each component via the command line

### Operators

```
oc apply -k operators
```

### MinIO (for s3 storage)

```
oc apply -k minio
```

### Enable User Monnitoring

```
oc apply -k observability
```

### Tempo Stack and OpenTelemetry (for distributed Tracing)
```
oc apply -k  tempoStack-coo/tempoStack
oc apply -k tempoStack-coo/observability-plugin
```
### Service Mesh Istio System
``` 
oc apply -k ossm/mesh
```
### Istio Gateway (with gateway-injection)
```
oc apply -k ossm/gateway-injection
```

### Kiali
```
oc apply -k kiali
```

### Sample `bookinfo` App (to test tracing and Kiali)
```
oc apply -k bookinfo
HOST=$(oc get route istio-ingressgateway -n prod-gateway -o jsonpath='{.spec.host}')
echo productpage URL: https://$HOST/productpage
```            

Bookinfo load generator
```
export INGRESSHOST=$(oc get route istio-ingressgateway -n prod-gateway -o=jsonpath='{.spec.host}')
cat ./bookinfo/bookinfo-traffic-gen/traffic-generator-configmap.yaml | ROUTE="https://${INGRESSHOST}/productpage" envsubst | oc -n bookinfo apply -f - 
oc apply -f ./bookinfo/bookinfo-traffic-gen/traffic-generator.yaml -n bookinfo
```

### GitOps (ArgoCD)

ArgoCD Applications are found in the `gitops` directory

Most components can be deployed all at once with the command

```
oc apply -k gitops
```

Once everything is up and running, manually install Kiali (WIP)
```
oc apply -k kiali
```

## Links to sections for more information. Please note that procedures may differ than from above

0. [Operator Installation](/00_OPERATORS.md)
1. [Installing OpenShift Service Mesh 3 components (Operator/Istio/Gateway)](/01_OSSM_SETUP.md)
2. [Observability Integration](/02_OBSERVABILITY.md)
3. [A Sample app to test your mesh configuration](/03_SAMPLE_APPS.md)
4. [(Optional) Kubernetes Gateway API setup for ingress](/04_GATEWAY_API)
5. [A condensed set of end to end steps, based on 0-3](/05_STEPS.md)

---
## Some useful commands

### restart every Deployment in a namespace
```bash
oc -n <namespace> get deploy -o name | xargs -r -L1 oc -n <namespace> rollout restart
```


---
TODO: 
- cert management
- ambient mode
- Vault integration for object storage credentials