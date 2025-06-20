---
num: 11
title: Java to Python Service Migration
status: "Draft"
authors:
  - "ruivieira"
tags:
  - "service"
  - "python"
  - "java"
  - "migration"
---

## Title

TrustyAI Service Migration from Java to Python

## Context and Problem Statement

The current TrustyAI service uses Java with Quarkus and provides explainability, fairness and drift monitoring for machine learning models in OpenShift/Kubernetes environments.
However, the machine learning ecosystem is mostly Python-based, with most AI safety libraries, explainability tools, data science kits, and model frameworks built in Python.
This includes traditional ML explainability and fairness metrics, plus generative AI safety tools, multi-modal assessment frameworks, and AI governance solutions.
The Java version creates barriers for integration with Python-native libraries and limits our ability to use the broader Python AI environment.

Additionally, maintaining both Java core and Python bindings creates complexity and extra maintenance work.

## Goals

* Provide a drop-in replacement Python service that maintains API compatibility with the existing Java version
* Use native Python AI tools for AI safety, including explainability, fairness metrics, generative AI safety, and multi-modal assessment
* Reduce maintenance work by removing the need for Java-Python bindings
* Keep compatibility with existing deployment patterns and integrations
* Enable easier extension and customisation of AI safety algorithms across traditional ML and generative AI
* Keep existing OpenShift/Kubernetes deployment features
* Remove the Java dependency

## Non-goals


* Change the external API contract or deployment model
* Build endpoints that are not actively used in production or downstream deployments (some endpoints may be omitted if they are not used in practice)
* Provide feature parity for all Java-specific internal code

## Current situation

The Java TrustyAI service provides a complete REST API with the following main endpoint categories:

**Implemented Endpoints:**
- Data Upload/Download (`/data/upload`, `/data/download`)
- Drift Metrics (`/metrics/drift/*` - ApproxKSTest, FourierMMD, KSTest, Meanshift)
- Fairness Metrics (`/metrics/group/fairness/*` - DIR, SPD)
- Explainers (`/explainers/global/*`, `/explainers/local/*` - LIME, SHAP, Counterfactual, PDP, TSSaliency)
- Identity Metrics (`/metrics/identity/*`)
- Consumer Endpoints (`/consumer/kserve/v2`)
- Service Metadata (`/info/*`)
- Legacy Endpoints (`/metrics/dir`, `/metrics/spd`)

**Python components:**
The Python service will be built using FastAPI and include:
- Basic service structure with health checks and Prometheus metrics
- Partial fairness metrics (DIR, SPD)
- Basic drift metrics
- Explainer endpoints (global and local)
- Data upload/download features
- Consumer endpoint for KServe v2 payloads
- Service metadata endpoints
- Complete algorithmic code for all metrics and explainers
- Full data storage and retrieval
- Complete KServe v2 payload processing
- Scheduled metrics computation
- Additional endpoints as needed for production use cases

## Proposal

### Migration Strategy

1. **Incremental Migration**: We will complete the Python service to achieve full API compatibility
2. **Drop-in Replacement**: We will ensure the Python service can be deployed as a direct replacement for the Java version
3. **Algorithmic Preservation**: We will maintain the same algorithmic behaviour for consistency
4. **Selective Focus**: We will focus on endpoints that are actively used in downstream deployments

### Technical Implementation

**Framework**: We will use FastAPI as it provides:
- Automatic OpenAPI documentation generation
- High performance async features
- Type hints and validation

**Endpoint Coverage**: We will build all endpoints documented in the [TrustyAI Service API Reference](https://trustyai-explainability.github.io/trustyai-site/main/trustyai-service-api-reference.html) except where otherwise noted, including:

- **DataUpload**: `POST /data/upload`
- **DownloadEndpoint**: `POST /data/download`
- **DriftMetrics**: Complete ApproxKSTest, FourierMMD, KSTest, and Meanshift
- **FairnessMetrics**: Full DIR and SPD with group fairness features
- **ExplainersGlobal**: LIME and PDP global explainers
- **ExplainersLocal**: LIME, SHAP, Counterfactual, and TSSaliency local explainers
- **IdentityEndpoint**: Identity metrics computation
- **ServiceMetadata**: Complete service information and metadata endpoints
- **MetricsInformation**: Metrics scheduling and information endpoints
- **LegacyEndpoints**: Keep backward compatibility with legacy metric endpoints

**Data Storage**: We will build equivalent data persistence using:
- HDF5 for data storage
- Pandas for data manipulation
- Prometheus client for metrics exposure
- MariaDB

**Deployment Compatibility**: We will keep compatibility with:
- OpenShift/Kubernetes deployments
- TrustyAI Operator management
- Existing service discovery and network patterns
- Health check endpoints (`/q/health/ready`, `/q/health/live`)
- Prometheus metrics endpoint (`/q/metrics`)

### Algorithmic Implementation

**Native Python Libraries**: We will use existing Python libraries where appropriate:
- scikit-learn for standard ML metrics
- SHAP library for SHAP explanations
- LIME library for LIME explanations
- NumPy/SciPy for statistical computations

**Custom Code**: We will build TrustyAI-specific algorithms to keep consistency with the Java version

**Testing**: We will ensure algorithmic parity through complete testing against Java results

## Alternatives Considered / Rejected

### Keep Dual Implementation
Continue maintaining both Java and Python versions was rejected due to:
- Increased maintenance work
- Complexity in keeping code synchronised
- Limited ability to use Python-native libraries in the Java service

### Hybrid Approach
Using JPype or similar Java-Python bridges was rejected due to:
- Complexity in deployment and container management
- Performance overhead from language bridges
- Limited ability to extend with pure Python libraries

### Gradual Feature Migration
A gradual migration with feature flags was rejected due to:
- Complexity in managing two codebases
- Potential for inconsistent behaviour
- Difficulty in keeping API compatibility

## Challenges

### Technical Challenges
- **Algorithmic Parity**: Ensuring Python code produces identical results to Java code
- **Performance**: Keeping comparable performance characteristics
- **Memory Management**: Handling large datasets efficiently in Python

### Operational Challenges
- **Deployment Transition**: Ensuring smooth transition for existing deployments
- **Documentation**: Updating all documentation and examples
- **Testing**: Complete testing to ensure feature parity
- **Monitoring**: Keeping observability and monitoring features

### Integration Challenges
- **Operator Compatibility**: Ensuring TrustyAI Operator continues to work with the Python service
- **Third-party Integrations**: Keeping compatibility with KServe, ModelMesh, and other integrations
- **API Compatibility**: Ensuring exact API compatibility for existing clients

## Dependencies

### Development Dependencies
- Completion of Python service algorithmic code
- Complete test suite development
- Performance benchmarking and optimisation
- Documentation updates

### Deployment Dependencies
- Container image updates
- Operator compatibility testing
- Downstream integration testing
- Migration documentation and tooling

### Environment Dependencies
- Python ML library environment stability
- FastAPI framework continued support
- Container base image security and maintenance

## Consequences if not completed

### Negative Consequences
- **Limited Extensibility**: Continued inability to easily integrate Python-native ML libraries
- **Maintenance Burden**: Ongoing complexity of keeping Java-Python bindings
- **Innovation Constraints**: Reduced ability to use Python ML research innovations
- **Community Barriers**: Difficulty for Python-focused contributors to contribute to the project

### Missed Opportunities
- **Environment Integration**: Lost opportunities to integrate with Python ML environment
- **Simplified Architecture**: Continued complexity from dual-language setup
- **Developer Experience**: Reduced developer productivity due to language barriers

## Success Criteria

The migration will be considered successful when:
1. **API Compatibility**: All documented endpoints are built with identical behaviour
2. **Performance Parity**: Performance characteristics match the Java version
3. **Deployment Compatibility**: Drop-in replacement capability is verified
4. **Algorithmic Consistency**: Results are equivalent to Java version
5. **Integration Testing**: All downstream integrations work without modification
6. **Documentation**: Complete documentation and migration guides are available 