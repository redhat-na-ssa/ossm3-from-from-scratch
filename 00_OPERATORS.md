
Install operators

```bash
oc apply -f operators     
```

Wait for operators to become available

```bash
until oc get pods -n openshift-operators | grep servicemesh-operator3 | grep Running; do echo "Waiting for servicemesh-operator3 to be running."; sleep 10;done
#until oc get pods -n openshift-operators | grep kiali-operator | grep Running; do echo "Waiting for kiali-operator to be running."; sleep 10;done
until oc get pods -n openshift-opentelemetry-operator | grep opentelemetry-operator | grep Running; do echo "Waiting for opentelemetry-operator to be running."; sleep 10;done
until oc get pods -n openshift-tempo-operator | grep tempo-operator | grep Running; do echo "Waiting for tempo-operator to be running."; sleep 10;done
until oc get pods -n openshift-cluster-observability-operator | grep observability-operator | grep Running; do echo "Waiting for observability-operator to be running."; sleep 10;done
```