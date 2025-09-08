
# Operator Installation

Before getting started, There are a number of operators that will need to be installed in order to implement 
OpenShift Service Mesh 3

## Operators

### Red Hat OpenShift Service Mesh 3

Red Hat OpenShift Service Mesh is a platform that provides behavioral insight and operational control over a service mesh, providing a uniform way to connect, secure, and monitor microservice applications

Red Hat OpenShift Service Mesh supports uniform application of a number of key capabilities across a network of services:

- Traffic Management - Control the flow of traffic and API calls between services, make calls more reliable, and make the network more robust in the face of adverse conditions.

- Service Identity and Security - Provide services in the mesh with a verifiable identity and provide the ability to protect service traffic as it flows over networks of varying degrees of trustworthiness.

- Policy Enforcement - Apply organizational policy to the interaction between services, ensure access policies are enforced and resources are fairly distributed among consumers. Policy changes are made by configuring the mesh, not by changing application code.

- Telemetry - Gain understanding of the dependencies between services and the nature and flow of traffic between them, providing the ability to quickly identify issues.

### Tempo Operator
 
Tempo is used for monitoring and troubleshooting microservices-based distributed systems, including:

- Distributed transaction monitoring
- Root cause analysis
- Performance / latency optimization

### Red Hat build of OpenTelemetry

Red Hat build of OpenTelemetry is a collection of tools, APIs, and SDKs. You use it to instrument, generate, collect, and export telemetry data (metrics, logs, and traces) for analysis in order to understand your software's performance and behavior. This operator was previously called Red Hat OpenShift distributed tracing data collection.

- Sidecar injection - annotate your pods and let the operator inject a sidecar.
- Managed upgrades - updating the operator will automatically update your OpenTelemetry collectors.
- Deployment modes - your collector can be deployed as sidecar, daemon set, or regular deployment.
- Service port management - the operator detects which ports need to be exposed based on the provided configuration.

### Kiali Operator

Kiali works with OpenShift Service Mesh to visualize the service mesh topology, to provide visibility into features like circuit breakers, request rates and more. It offers insights about the mesh components at different levels, from abstract Applications to Services and Workloads.

See https://www.kiali.io to read more.

### Cluster Observability Operator

Cluster Observability Operator is an operator to easily setup and manage various observability tools.

We will use it for the tracing console plugin

- Setup multiple Highly Available Monitoring stack using Prometheus, Alertmanager and Thanos Querier
- Customizable configuration for managing Prometheus deployments
- Customizable configuration for managing Alertmanager deployments
- Customizable configuration for managing Thanos Querier deployments
- Setup console plugins
- Setup korrel8r
- Setup Perses
- Setup Cluster Health Analyzer

You can install all of the above mentioned operators manually via Operator Hub in the OpenShift Web Console, or by running the following command.

```bash
oc apply -f operators     
```

Wait for operators to become available

```bash
until oc get pods -n openshift-operators | grep servicemesh-operator3 | grep Running; do echo "Waiting for servicemesh-operator3 to be running."; sleep 10;done
until oc get pods -n openshift-operators | grep kiali-operator | grep Running; do echo "Waiting for kiali-operator to be running."; sleep 10;done
until oc get pods -n openshift-opentelemetry-operator | grep opentelemetry-operator | grep Running; do echo "Waiting for opentelemetry-operator to be running."; sleep 10;done
until oc get pods -n openshift-tempo-operator | grep tempo-operator | grep Running; do echo "Waiting for tempo-operator to be running."; sleep 10;done
until oc get pods -n openshift-cluster-observability-operator | grep observability-operator | grep Running; do echo "Waiting for observability-operator to be running."; sleep 10;done
```

---
[Back to main README](/README.md)/[Observability Config](/01_OBSERVABILITY.md)