---
num: 3 # allocate an id when the draft is created
title: TrustyAI Service Deployment using Operator pattern
status: "Draft" # One of Draft, Accepted, Rejected
authors:
  - "ruivieira"
  - "danielezonca" # One item for each author, as github id or "firstname lastname"
tags:
  - "service" # e.g. service, python, java, etc
---

## Title

TrustyAI Service Deployment using Operator pattern

## Context and Problem Statement

The [TrustyAI Service](https://github.com/trustyai-explainability/trustyai-explainability/tree/main/explainability-service) can be deployed manually as a standalone container or via [ODH-manifest](https://github.com/opendatahub-io/odh-manifests/blob/master/trustyai-service/) as part of ODH KfDef. Both cases have limitations: a plain Deployment is error prone for the users (some parameters are mandatory) and the ODH-manifest contains [some hacks](https://github.com/opendatahub-io/odh-manifests/blob/master/trustyai-service/default/trustyai-deployment.yaml#L69-L87).
A Kubernetes operator would provide a simple and consistent way to deploy and manage the TrustyAI service.

In addition to this, the deployment and the storage (PVC for now) must be created into a user owned namespace to give users full control and prevent security issues. An operator can enforce this.

## Goals

* Automate the deployment, management, and maintenance of the TrustyAI service.
* Reduce manual errors and increase consistency in deployments.
* Help updating the TrustyAI service.

## Non-goals

* Implementing mechanisms that perform actions unrelated with the lifecycle of the TrustyAI service (create, upgrade, monitor, etc)..
* In the initial stage, distribution via OperatorHub is not a goal. This may be considered in the future.

## Current situation

Currently, TrustyAI service deployments are done manually or through scripts that do not fully take advantage of Kubernetes. This can be inefficient and lead to errors introduced by manual steps.

As an example, the TrustyAI service needs to update ModelMesh's configuration to add the TrustyAI service as a new endpoint. This is currently done via a deployment-time script that patches the ModelMesh configuration. This could be done automatically by the Operator.

Although TrustyAI's deployment needs to configure a considerable number of resources (_e.g._ `Deployment`, `Service`, `ConfigMap`, `Route`, `ServiceMonitor`), the actual configuration options available for a custom TrustyAI deployment are limited. This means that a custom TrustyAI Custom Resource Definition (CRD) would be quite simple, and the Operator would be able to handle the creation and management of the required resources.

## Proposal

We propose to use a stand-alone TrustyAI Kubernetes Operator which would create and manage the required Deployment, Service, ConfigMap, Route, and ServiceMonitor resources based on a simple Custom Resource while keeping the state consistent with the desired one [^1].

[^1]: Initial implementation at https://github.com/ruivieira/trustyai-service-operator

### Custom Resource

An example of a custom resource is:

```yaml
apiVersion: trustyai.opendatahub.io/v1alpha1
kind: TrustyAIService
metadata:
  name: trustyai-service-example
  
spec:
  storage:
    format: "PVC"
    folder: "/inputs"
    pv: "mypv"
    size: "1Gi"
  data:
    filename: "data.csv"
    format: "CSV"
  trustyaiMetrics:
    schedule: "15s"
status:
  phase: …
  replicas: …
  conditions:
  - type: Ready
    …
  - type: ModelMeshReady 
    status: "True"
    lastTransitionTime: …
    reason: ModelMeshHealthy
    message: ModelMesh is running and healthy.
  - type: StorageReady
    status: "True"
    lastTransitionTime: …
    reason: StorageHealthy
    message: Storage system is functioning correctly.
    lastUpdateTime: …
```

In this example:

* `replicas` is an optional field that specifies the number of replicas of the TrustyAI service that you want to run. If not provided, the default is one replica.
* `storage` is a mandatory field that specifies the storage details. It has two nested fields:
  * `format` - the storage format, (example: a Persistent Volume Claim (PVC)).
  * `folder` - the folder path where data is stored.
  * `pv` - the name of the Persistent Volume (PV) to use (already existing).
  * `size` - the size of the PV to use (example: 1Gi).
* data is a mandatory field that specifies the data details. It has two nested fields:
  * `filename` - the suffix of the file that the service uses for data.
  * `format` - the format of the data file (example:  a CSV file).
* `trustyaiMetrics` is a mandatory field that specifies the metrics details. It has one nested field:
  * `schedule` - the schedule for metrics calculation, (example: every 5 seconds).
* `status` - as part of the reconciliation process, the operator will add additional conditions, apart from the standard ones, to the custom resource to indicate the status of the deployment. These conditions will be:
  * `ModelMeshReady`, which indicates that the ModelMesh Serving component is running.
  * `StorageReady`, which indicates that the storage component is running.

The storage, data and metrics keys consist of the only mandatory configuration fields for the TrustyAI service, at the moment. Future configuration keys can be added to the custom resource as needed.

The proposed `apiVersion` and `kind` are `trustyai.opendatahub.io/v1alpha1` and `TrustyAIService`, respectively.

### ModelMesh Serving Integration

The operator also ensures the correct configuration of the ModelMesh Serving component. Once the TrustyAI Service is deployed and reachable, the operator will patch the ModelMesh Serving configuration to include a custom payload processor and it will be configured to point to the consumer endpoint of the deployed TrustyAI Service.

The processor configuration is embedded in a Kubernetes ConfigMap and follows the format:


```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: model-serving-config
data:
  config.yaml: |
    payloadProcessors: http://trustyai-service.$NAMESPACE.svc.cluster.local/consumer/kserve/v2
```

In this configuration, `$NAMESPACE` is replaced by the Operator with the namespace where the TrustyAI Service and ModelMesh Serving are deployed ensuring that ModelMesh sends payloads correctly to the TrustyAI Service.

### Monitoring (Prometheus)

The TrustyAI Operator also creates a `ServiceMonitor` object which defines the services to be monitored by Prometheus. The `ServiceMonitor` will have the following configuration:


```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: trustyai-metrics
  labels:
    modelmesh-service: modelmesh-serving
spec:
  endpoints:
    - interval: 4s
      path: /q/metrics
      honorLabels: true
      honorTimestamps: true
      scrapeTimeout: 3s
      bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
      targetPort: 8080
      scheme: http
      params:
   	 'match[]':
 		 - '{__name__= "trustyai_spd"}'
 		 - '{__name__= "trustyai_dir"}'
      metricRelabelings:
   	 - action: keep
 		 regex: trustyai_.*
 		 sourceLabels:
   		 - __name__
  selector:
    matchLabels:
      app.kubernetes.io/name: trustyai-service
```


The `ServiceMonitor` object targets the TrustyAI Service and specifies how Prometheus should scrape metrics from the service, which includes the path to the metrics endpoint (`/q/metrics`), the interval at which it should scrape the metrics (every 4 seconds), and the type of metrics it should scrape (metrics with names that start with `trustyai_`).
The selector would also be updated to match the labels of the TrustyAI Service from the Custom Resource.
The scrape interval and metrics names could potentially also be configurable via the custom resource (with the current values as defaults).


A possibility for the service monitor customization is the inclusion of custom values using a nested `ref:` field for `serviceMonitoring`. This would allow users to specify a custom configuration for the `ServiceMonitor` object. For example:


```yaml
apiVersion: trustyai.opendatahub.io/v1
kind: TrustyAIService
metadata:
  name: trustyai-service-example
spec:
  ...
  serviceMonitoring:
    ref:
      apiVersion: monitoring.coreos.com/v1
      kind: ServiceMonitor
      ...
      spec:
   	 endpoints:
 		 - interval: 15s
```

If such configuration is not provided, the operator will use the default configuration.

### Route

If deployed on OpenShift, the Operator will also create a `Route` object to expose the TrustyAI Service to external clients. The `Route` object will have the following configuration:


```yaml
kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: trustyai
  labels:
    app: trustyai
    app.kubernetes.io/name: trustyai-service
    app.kubernetes.io/part-of: trustyai
    app.kubernetes.io/version: 0.1.0
spec:
  to:
    kind: Service
    name: trustyai-service
  port:
    targetPort: http
  tls: null
```


Note that TrustyAI isn't currently implementing HTTPS endpoints, so the `tls` field will be set to `null` for now. Once HTTPS is implemented, the `tls` field will be updated to include the TLS configuration.

### Storage

The TrustyAI service requires storage to store inference data. Upon CR deployment, the operator will create a `PersistentVolumeClaim` object to request storage for the TrustyAI Service. The `PersistentVolumeClaim` object will have the following configuration:

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: trustyai-service-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  volumeMode: Filesystem
  storageClassName: ""
```

and bind it to the TrustyAI Service deployment and supplied PV.
The PVC will be created in the same namespace as the TrustyAI Service is being deployed.

If multiple CRs are deployed in the same namespace, each will have its own PVC.
The PVC and PV naming rule is respectively `${CR_NAME}-pvc` and `${CR_NAME}-pv`, where `$CR_NAME` is the name of the CR.

### Custom Image Configuration using ConfigMap

If a custom image is required for the TrustyAI service (_e.g._ for development or testing), you can configure the operator to use custom images by creating a `ConfigMap` in the operator's namespace.
The operator only checks the `ConfigMap` at deployment, so changes made afterward won't trigger a redeployment of services.

An example of a ConfigMap that specifies a custom image:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: trustyai-service-operator-config
data:
  trustyaiServiceImageName: 'quay.io/mycustomrepo/mycustomimage'
  trustyaiServiceImageTag: 'v1.0.0'
```

After the ConfigMap is applied, the operator will use the image name and tag specified in the `ConfigMap` for the CR deployment.

Since this functionality is mainly for development and testing, if you want to use a different image or tag after deployment, you'll need to update the `ConfigMap` and redeploy the operator to have the changes take effect. The running TrustyAI services won't be redeployed automatically. To use the new image or tag, you'll need to delete and recreate the TrustyAIService resources.

### Testing

The testing and CI of the TrustyAI Operator will be performed using the following approaches:

* Unit tests for the Operator code, to ensure that the Operator's functionality is correct.
* Integration tests using [envtest](https://book.kubebuilder.io/reference/envtest.html) to ensure that the Operator is correctly deployed and configured. The Kuttl tests will, for instance, ensure that:
  * The state is correctly updated when the Custom Resource is updated.
  * Routes and ServiceMonitors are correctly created.
  * ModelMesh Payload Processors are correctly configured.
* End-to-End (E2E) tests, by integrating with the work already being implemented with the [TrustyAI E2E tests](https://github.com/trustyai-explainability/trustyai-explainability/tree/main/e2e_tests)

## Threat Model

* No other threats additionally to the ones common to any operators themselves, which include misconfiguration of the operator, security vulnerabilities in the operator code or in the created resources.

## Challenges

* Go and Kubernetes/OpenShift knowledge required to develop the Operator.

## Dependencies

None

## Consequences if not completed

If not completed, we will continue with the manual deployment and management of the TrustyAI service which would make it harder to scale and update the service.
