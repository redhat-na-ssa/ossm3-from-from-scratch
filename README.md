# ossm3-from-from-scratch
Following official RH documentation
---
Tested on
```bash
Client Version: 4.19.7
Kustomize Version: v5.5.0
Server Version: 4.18.22
Kubernetes Version: v1.31.11
```
## Links to sections

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
Note: It may be better to install observability components first then OSSM3

---
TODO: 
- Cluster Observability Operator for Tracing
- cert management
- deploy travel agency app
- ambient mode
- quickstart component that uses GitOps
- Vault integration for object storage credentials