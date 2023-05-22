---
num: 3 # allocate an id when the draft is created
title: TrustyAI Service Deployment using Operator
status: "Draft" # One of Draft, Accepted, Rejected
authors:
  - "ruivieira" # One item for each author, as github id or "firstname lastname"
tags:
  - "service" # e.g. service, python, java, etc
---

## Title

TrustyAI Service Deployment using Operator

## Context and Problem Statement

The [TrustyAI Service](https://github.com/trustyai-explainability/trustyai-explainability/tree/main/explainability-service) is currently deployed manually, leading to potential inconsistencies and errors.
A Kubernetes operator would provide a simple and consistent way to deploy and manage the TrustyAI service.

## Goals

1. Automate the deployment, management, and maintenance of the TrustyAI service.
2. Reduce manual errors and increase consistency in deployments.
3. Help updating of the TrustyAI service.

## Non-Goals

1. Implementing mechanisms that perform actions other than the deployment, management, and maintenance of the TrustyAI service.

## Current Situation

Currently, TrustyAI service deployments are done manually or through scripts that do not fully take advantage of Kubernetes. This can be inneficent and lead to errors introduced by manual steps.
As an example, the TrustyAI service needs to update ModelMesh's configuration to add the TrustyAI service as a new endpoint. This is currently done via a deployment-time script that patches the ModelMesh configuration. This could be done automatically by the Operator.

Althought TrustyAI's deployment needs to configure a considerable number of resources (_e.g._ `Deployment`, `Service`, `ConfigMap`, `Route`, `ServiceMonitor`), the actual configuration options available for a custom TrustyAI deployment are limited. This means that the a custom TrustyAI Custom Resource Definition (CRD) would be quite simple, and the Operator would be able to handle the creation and management of the required resources.

## Proposal

We propose to use a stand-alone TrustyAI Kubernetes Operator which would create and manage the required `Deployment`, `Service`, `ConfigMap`, `Route`, and `ServiceMonitor` resources based on a simple Custom Resource while keeping the state consistent with the desired one.

### Custom Resource

An example of a custom resource is:

```yaml
apiVersion: trustyai.opendatahub.io/v1
kind: TrustyAIService
metadata:
  name: trustyai-service-example
  namespace: default
spec:
  replicas: 1
  image: quay.io/trustyaiservice/trustyai-service
  tag: v1.0
  storage:
    format: "PVC"
    folder: "/inputs"
  data:
    filename: "data.csv"
    format: "CSV"
  metrics:
    schedule: "5s"
```

In this example:

1. `replicas` is an optional field that specifies the number of replicas of the TrustyAI service that you want to run. If not provided, the default is one replica.

2. `image` and `tag` are optional fields that allow you to specify a custom image and tag for the TrustyAI service. If not provided, the default is `quay.io/trustyaiservice/trustyai-service:latest`.

3. `storage` is a mandatory field that specifies the storage details. It has two nested fields:
   - `format` - the storage format, (example: a Persistent Volume Claim (PVC)).
   - `folder` - the folder path where data is stored.

4. `data` is a mandatory field that specifies the data details. It has two nested fields:
   - `filename` - the suffix of the file that the service uses for data.
   - `format` - the format of the data file (example:  a CSV file).

5. `metrics` is a mandatory field that specifies the metrics details. It has one nested field:
   - `schedule` - the schedule for metrics collection, (example: every 5 seconds).


The storage, data and metrics keys consist of the only mandatory configuration fields for the TrustyAI service, at the moment. Future configuration keys can be added to the custom resource as needed.

The proposed `apiVersion` and `kind` are `trustyai.opendatahub.io/v1` and `TrustyAIService`, respectively.

### ModelMesh Serving Integration

The operator also ensures the correct configuration of the ModelMesh Serving component. Once the TrustyAI Service is deployed and reachable, the operator will patch the ModelMesh Serving configuration to include a custom payload processor and it will be configured to point to the consumer endpoint of the deployed TrustyAI Service.

The processor configuration is embedded in a Kubernetes `ConfigMap` and follows the format:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: model-serving-config
  namespace: default
data:
  config.yaml: |
    payloadProcessors: http://trustyai-service.$NAMESPACE/consumer/kserve/v2
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

### Threat Model

No other threats additionally to the ones common to any operators themselves, which include misconfiguration of the operator, security vulnerabilities in the operator code or in the created resources.

## Challenges

1. Go and Kubernetes/OpenShift knowledge required to develop the Operator.

## Dependencies

1. Operator Lifecycle Manager (OLM) for installing and managing the Operator.

## Consequences if not completed

If not completed, we will continue with the manual deployment and management of the TrustyAI service which would make it harder to scale and update the service.