---
num: 10
title: TrustyAI SDK
status: "Draft"
authors:
  - "ruivieira"
tags:
  - "api"
  - "trustyai"
  - "sdk"
---

## Table of Contents

- [1. Title](#1-title)
- [2. Context and Problem Statement](#2-context-and-problem-statement)
- [3. Goals](#3-goals)
- [4. Non-goals](#4-non-goals)
- [5. Current situation](#5-current-situation)
- [6. Proposal](#6-proposal)
  - [6.1. TrustyAI SDK](#61-trustyai-sdk)
  - [6.2. Architecture Overview](#62-architecture-overview)
  - [6.3. Core API Structure](#63-core-api-structure)
    - [6.3.1. Provider Overview](#631-provider-overview)
  - [6.4. Data Models and Schemas](#64-data-models-and-schemas)
    - [6.4.1. Core Data Models](#641-core-data-models)
      - [6.4.1.1. Dataset Abstraction](#6411-dataset-abstraction)
      - [6.4.1.2. Request/Response Models](#6412-requestresponse-models)
    - [6.4.2. Common Provider API Arguments](#642-common-provider-api-arguments)
      - [6.4.2.1. Evaluation Provider Scope](#6421-evaluation-provider-scope)
      - [6.4.2.2. Red Teaming Provider Scope](#6422-red-teaming-provider-scope)
      - [6.4.2.3. Unified Provider Configuration](#6423-unified-provider-configuration)
    - [6.4.3. Provider-Specific Schemas](#643-provider-specific-schemas)
    - [6.4.4. Validation and Serialisation](#644-validation-and-serialisation)
    - [6.4.5. Configuration Schema](#645-configuration-schema)
    - [6.4.6. Error Handling Schema](#646-error-handling-schema)
  - [6.5. CLI Interface](#65-cli-interface)
    - [6.5.1. Consistent Command Structure](#651-consistent-command-structure)
    - [6.5.2. Command Comparison](#652-command-comparison)
    - [6.5.3. Advanced CLI Features](#653-advanced-cli-features)
  - [6.6. Provider Architecture](#66-provider-architecture)
    - [6.6.1. Provider Interface Design](#661-provider-interface-design)
    - [6.6.2. Example: Evaluation Provider](#662-example-evaluation-provider)
    - [6.6.3. Validators](#663-validators)
    - [6.6.4. Example: Red Team Provider *(Not Implemented)*](#664-example-red-team-provider-not-implemented)
  - [6.7. Usage Patterns](#67-usage-patterns)
  - [6.8. TrustyAI Service](#68-trustyai-service)
    - [6.8.1. Component Interaction Flow](#681-component-interaction-flow)
- [7. Threat model](#7-threat-model)
- [8. Alternatives Considered / Rejected](#8-alternatives-considered--rejected)
- [9. Challenges](#9-challenges)
- [10. Dependencies](#10-dependencies)
- [11. Consequences if not completed](#11-consequences-if-not-completed)

## 1. Title

TrustyAI SDK

## 2. Context and Problem Statement

TrustyAI currently lacks a unified, consistent API across its various services and components. This fragmentation complicates integration, increases the learning curve for developers, and may lead to inconsistent implementations. A universal API would provide a standardised interface for all TrustyAI functionality, enabling more cohesive AI safety capabilities.

Additionally, TrustyAI's core algorithms are currently built on Java, which creates several challenges:

- **Contribution barriers**: The Java-based core limits contributions from the broader AI/ML community, which predominantly works in Python
- **Language bridging complexity**: Integrating Python-based workflows with Java core algorithms requires complex bridging mechanisms and introduces additional maintenance overhead
- **Third-party library limitations**: The Java ecosystem has fewer specialised AI safety and ML libraries compared to Python, making it difficult to leverage community innovations in, for instance, explainability, bias detection, and evaluation frameworks

**Migration Strategy**: The new Python-based TrustyAI service will replace the existing Java service. The TrustyAI SDK serves as the core library that the service builds upon, providing a clear separation between the reusable components (SDK) and the deployment-specific service layer.

## 3. Goals

- Establish a consistent interface across all TrustyAI services and components
- Simplify integration for developers by providing a unified access pattern
- Structure capabilities into logical "scopes" of AI safety (explainability, evaluation, guardrails, etc.)
- Enable extensibility for future capabilities without breaking changes
- Improve developer experience and reduce onboarding time
- Enable consistent error handling and logging
- Target multiple platforms (local, kubernetes) with a single API

## 4. Non-goals

- Rewriting all existing implementations immediately
- Rewriting the component APIs (e.g. LMEval)
- Supporting legacy systems that cannot be reasonably adapted
- Replacing ad-hoc APIs if it is not possible to reasonably adapt them

## 5. Current situation

TrustyAI offers various capabilities across different components, each with its own interface patterns and access methods. This creates inconsistencies in how developers interact with the system and complicates integration efforts. There is no unified model for representing different AI safety concerns such as explainability, guardrails, and evaluation, especially on different target platforms (local, kubernetes).

## 6. Proposal

Develop a universal API for TrustyAI based on a two-tier architecture consisting of a standalone SDK and services that build upon it:

### 6.1. TrustyAI SDK

A standalone Python package (`trustyai`) that provides:

1. **Provider-centric architecture** offering abstractions for different AI safety scopes:
   - Explainability
   - Evaluation
   - Guardrails
   - Monitoring
   - Bias detection
   - Red teaming
   - _etc._

2. **Core features**:
   - Abstract _Provider_ interfaces that encapsulate universal features for each AI safety scope
   - Common data models and request/response formats
   - Pluggable implementation system allowing multiple backends per provider
   - Unified configuration patterns
   - Error handling and logging frameworks
   - Extension mechanisms for future capabilities
   - **CLI interface** providing command-line access to all provider functionality

3. **Distribution**:
   - Installable via `pip install trustyai`
   - Provides programmatic access to all AI safety capabilities
   - Can be used directly in Python applications or as a core foundation for services

4. **Broad target support**:
   - **CLI**: Command-line interface for direct terminal usage
   - **Library**: Programmatic Python API for application integration
   - **Jupyter notebooks**: Interactive data science and research workflows
   - **Kubernetes components**: Cloud-native AI safety services

**Service Layer**: The TrustyAI REST service imports and uses the SDK to provide web-based access to all capabilities. This service will replace the existing Java-based TrustyAI service, providing the same functionality through a Python implementation that leverages the broader Python AI ecosystem.

### 6.2. Architecture Overview

```mermaid
graph TD
    A[TrustyAI CLI] --> |"import trustyai"| C[TrustyAI SDK]
    B[Jupyter Notebooks] --> |"import trustyai"| C
    D[TrustyAI Service] --> |"import trustyai"| C

    subgraph "TrustyAI Services Layer"
        A[CLI Interface]
        B[Jupyter Notebooks<br/>- Interactive analysis<br/>- Research workflows<br/>- Data exploration]
        D[TrustyAI Service<br/>- Evaluation endpoints<br/>- Explainability endpoints<br/>- Provider validation<br/>- Job management]
    end

    C --> E[Provider Interface Layer]

    subgraph "TrustyAI SDK: pip install trustyai"
        C[TrustyAI Core]
        E[Provider Interface]
        F[Evaluation Provider<br/>- Metrics calculation<br/>- Model assessment<br/>- Performance analysis]
        G[Explainability Provider<br/>- Feature importance<br/>- Model interpretation<br/>- Prediction explanations]
        H[Guardrails Provider<br/>- Safety constraints<br/>- Input/output validation<br/>- Policy enforcement]
        I[Monitoring Provider<br/>- Model drift<br/>- Performance tracking<br/>- Alert management]
        J[Bias Detection Provider<br/>- Fairness metrics<br/>- Bias assessment<br/>- Demographic analysis]
    end

    E --> F
    E --> G
    E --> H
    E --> I
    E --> J

    subgraph "Provider Implementations"
        K[LM-Eval Harness Local<br/>Local execution]
        L[LM-Eval Harness K8s<br/>Kubernetes execution]
        M[Garak Local<br/>Local execution]
        N[Garak K8s<br/>Kubernetes execution]
        O[Fairlearn Local<br/>Local execution]
        P[Fairlearn K8s<br/>Kubernetes execution]
        Q[Integrated Gradients]
        R[Nemo Guardrails]
        S[FMS Guardrails]
    end

    F --> K
    F --> L
    F --> M
    F --> N
    G --> O
    G --> P
    G --> Q
    H --> R
    H --> S

    subgraph "Datasets"
        V[Text File Provider]
        W[SQLite Provider]
        X[MariaDB Provider]
        Y[S3 Provider]
    end

    F -.-> V
    F -.-> W
    F -.-> X
    F -.-> Y
    G -.-> V
    G -.-> W
    G -.-> X
    G -.-> Y
    J -.-> V
    J -.-> W
    J -.-> X
    J -.-> Y

    subgraph "Deployment Options"
        T[Local]
        U[Kubernetes]
    end

    D --> T
    D --> U
```

### 6.3. Core API Structure

#### 6.3.1. Provider Overview

The TrustyAI SDK employs a provider-based architecture where each AI safety domain is represented by a standardised `Provider` interface. This design ensures consistent APIs while allowing flexible implementation backends.

**Provider Design Principles:**

1. **Provider-based design**: Each AI safety scope will be represented by a `Provider` interface that defines the capabilities for that domain.
2. **Pluggable implementations**: Providers can have multiple implementations that can be swapped based on requirements.
3. **Common data models**: Shared data structures across providers for consistent representation.
4. **Unified configuration**: Common configuration patterns across all providers.
5. **Package independence**: Can be used directly in Python applications without requiring additional services.
6. **Platform-specific implementations**: Each provider must have at least **one** platform-specific implementation (`LocalExecutor` or `KubernetesExecutor` for now), but can (and should) support both for maximum deployment flexibility (see next point).

**Platform-Specific Provider Implementations:**

All providers must implement at least **one** platform-specific execution target to ensure broad deployment compatibility:

1. **LocalExecutor Implementation**: Executes operations directly on the local machine where the SDK is running. This implementation:
   - Handles direct process execution
   - Manages local file system interactions
   - Validates local dependencies and system requirements
   - Designed for development workflows and single-machine deployments

2. **KubernetesExecutor Implementation**: Deploys operations as resources within a Kubernetes cluster. This implementation:
   - Transforms provider requests into Kubernetes Custom Resources or Job specifications
   - Manages cluster connectivity and authentication
   - Handles resource allocation, scaling, and job lifecycle management
   - Designed for production deployments and scalable workloads

**Implementation Requirements:**

- **Minimum requirement**: Each provider must have at least one of `LocalExecutor` or `KubernetesExecutor`
- **Recommended approach**: Implement both `LocalExecutor` and `KubernetesExecutor` for maximum flexibility
- **Consistent API**: Both implementations must expose identical interfaces while handling platform-specific details internally
- **Validation specialisation**: Each implementation includes platform-specific validators (_e.g._, local dependency checks vs. cluster connectivity validation)

**Example Implementation Pattern:**

(_All code examples are not final and are only intended to illustrate the patterns._)

```python
# Evaluation provider with both platform implementations
class EvaluationProvider(BaseProvider):
    def __init__(self, implementation: str, execution_mode: str = "local", **config):
        super().__init__(implementation, execution_mode, **config)

        # Platform-specific implementation selection
        if self.execution_mode == "local":
            self.executor = LocalEvaluationExecutor(implementation, config)
        elif self.execution_mode == "kubernetes":
            self.executor = KubernetesEvaluationExecutor(implementation, config)
        else:
            raise ValueError(f"Unsupported execution mode: {execution_mode}")

    def evaluate(self, model, tasks, parameters):
        # Universal API - platform differences handled internally
        return self.executor.execute_evaluation(model, tasks, parameters)

class LocalEvaluationExecutor:
    """Local machine execution for evaluation tasks."""

    def execute_evaluation(self, model, tasks, parameters):
        # Direct CLI invocation and process management
        # Local file system handling
        # Environment validation
        pass

class KubernetesEvaluationExecutor:
    """Kubernetes cluster execution for evaluation tasks."""

    def execute_evaluation(self, model, tasks, parameters):
        # Custom Resource generation
        # Job deployment and monitoring
        # Cluster resource management
        pass
```

This architecture ensures that:
- **Development Flexibility**: Providers can start with `LocalExecutor` for rapid prototyping
- **Consistent Experience**: Users interact with the same API regardless of deployment target
- **Implementation Shield**: Platform complexity is abstracted away from the user

**Considered Examples of Providers:**

- **EvaluationProvider**: Model evaluation and benchmarking
- **ExplainabilityProvider**: Model interpretation, feature importance, and prediction explanations
- **BiasDetectionProvider**: Fairness metrics, bias assessment, and demographic analysis
- **GuardrailsProvider**: Safety constraints, input/output validation
- **MonitoringProvider**: Model drift detection, performance tracking, and alert management
- **RedTeamingProvider**: Adversarial testing, security assessment, and vulnerability detection
- **AlignmentProvider**: Value alignment assessment, reward model validation, and goal specification testing

**Provider Architecture Diagram:**

```mermaid
graph TD
    A[TrustyAI Client] --> B[Provider Factory]

    B --> C[EvaluationProvider]
    B --> D[ExplainabilityProvider]
    B --> E[BiasDetectionProvider]
    B --> F[GuardrailsProvider]
    B --> G[MonitoringProvider]

    C --> C1[LM-Eval Harness]
    C --> C2[RAGAS]

    D --> D1[SHAP]
    D --> D2[LIME]

    E --> E1[Fairlearn]
    E --> E2[Custom Metrics]

    F --> F1[Nemo Guardrails]
    F --> F2[Custom Policies]

    G --> G1[Data Drift Detection]
    G --> G2[Performance Monitor]

    subgraph "Execution Modes"
        X1[Local]
        X2[Kubernetes]
    end

    subgraph "Common Infrastructure"
        Y1[Validation]
        Y2[Configuration]
        Y3[Error Handling]
        Y4[Data Models]
    end

    C -.-> X1
    C -.-> X2
    D -.-> X1
    D -.-> X2
    E -.-> X1
    F -.-> X1
    F -.-> X2
    G -.-> X1
    G -.-> X2

    C -.-> Y1
    D -.-> Y1
    E -.-> Y1
    F -.-> Y1
    G -.-> Y1
```

**Platform Implementation Requirements:**

Each provider implementation must support at least one execution mode (Local or Kubernetes), with the recommendation to support both for maximum deployment flexibility. The diagram above shows example execution mode support, but actual implementations may vary based on technical feasibility and use case requirements.

**Universal Provider Interface:**

All providers inherit from a base `BaseProvider` class that defines common functionality:

```python
from abc import ABC, abstractmethod
from typing import Dict, Any, Optional
from trustyai.models import TrustyAIRequest, TrustyAIResponse

class BaseProvider(ABC):
    """Base provider interface for all TrustyAI capabilities."""

    def __init__(self, implementation: str, execution_mode: str = "local", **config):
        self.implementation = implementation
        self.execution_mode = execution_mode
        self.config = config
        self.validator = self._get_validator()

    @abstractmethod
    def _get_validator(self) -> 'BaseValidator':
        """Get the appropriate validator for this provider implementation."""
        pass

    @abstractmethod
    def execute(self, request: TrustyAIRequest) -> TrustyAIResponse:
        """Execute the provider operation."""
        pass

    def validate(self) -> 'ValidationResult':
        """Validate system readiness for this provider."""
        return self.validator.validate()

    def _prepare_execution(self, request: TrustyAIRequest) -> Dict[str, Any]:
        """Prepare execution parameters from request."""
        # Common preparation logic
        pass
```

**Provider Usage Pattern:**

```python
from trustyai.providers import EvaluationProvider, ExplainabilityProvider

# Evaluation with local execution
eval_provider = EvaluationProvider(
    implementation="lm-evaluation-harness",
    execution_mode="local"
)
eval_result = eval_provider.evaluate(model, tasks, parameters)

# Explainability with external API (configured within local execution)
explain_provider = ExplainabilityProvider(
    implementation="shap",
    execution_mode="local",
    config={
        "api_endpoint": "https://api.trustyai.com",
        "api_key": "your-api-key"
    }
)
explanation = explain_provider.explain(model, input_data)

# All providers support the same configuration patterns
providers = [eval_provider, explain_provider]
for provider in providers:
    validation_result = provider.validate()
    if validation_result.is_valid:
        print(f"✅ {provider.__class__.__name__} ready")
    else:
        print(f"❌ {provider.__class__.__name__} issues found")
```

**Key Components:**
- Provider interfaces and factory methods
- Common request/response formats
- Error handling patterns
- Versioning strategy
- Extension mechanisms

### 6.4. Data Models and Schemas

The TrustyAI SDK standardises data representation across all providers through a unified schema architecture built on Pydantic models. This ensures type safety, validation, and consistent serialisation across all AI safety capabilities.

#### 6.4.1. Core Data Models

**Base Models:**

```python
from pydantic import BaseModel, Field
from typing import Optional, Dict, Any, List, Union
from datetime import datetime
from enum import Enum

class ExecutionMode(str, Enum):
    LOCAL = "local"
    KUBERNETES = "kubernetes"

class JobStatus(str, Enum):
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"
    CANCELLED = "cancelled"

class TrustyAIMetadata(BaseModel):
    """Common metadata across all TrustyAI operations."""
    job_id: str = Field(description="Unique identifier for the operation")
    provider_type: str = Field(description="Type of provider (evaluation, explainability, etc.)")
    implementation: str = Field(description="Specific implementation used")
    execution_mode: ExecutionMode = Field(description="Execution environment")
    created_at: datetime = Field(default_factory=datetime.utcnow)
    completed_at: Optional[datetime] = None
    status: JobStatus = Field(default=JobStatus.PENDING)
    version: str = Field(description="TrustyAI version used")

class ModelReference(BaseModel):
    """Standardised model reference format."""
    identifier: str = Field(description="Model identifier (HF path, file path, API endpoint)")
    type: str = Field(description="Model type (huggingface, local, api, etc.)")
    version: Optional[str] = None
    metadata: Dict[str, Any] = Field(default_factory=dict)

    class Config:
        schema_extra = {
            "examples": [
                {"identifier": "hf/microsoft/DialoGPT-medium", "type": "huggingface"},
                {"identifier": "/path/to/model.pkl", "type": "local"},
                {"identifier": "https://api.openai.com/v1", "type": "api"}
            ]
        }

class DatasetReference(BaseModel):
    """Standardised dataset reference format."""
    identifier: str = Field(description="Dataset identifier or path")
    type: str = Field(description="Dataset type (file, huggingface, synthetic, etc.)")
    format: Optional[str] = Field(description="Data format (json, csv, parquet, etc.)")
    schema: Optional[Dict[str, Any]] = Field(description="Dataset schema information")
    metadata: Dict[str, Any] = Field(default_factory=dict)

#### 6.4.1.1. Dataset Abstraction

The TrustyAI SDK implements a unified Dataset abstraction that provides a consistent interface for accessing data from various sources while using pandas DataFrame as the universal data format. This abstraction allows providers to work with data from databases, files, cloud storage, and other sources without requiring knowledge of the underlying storage implementation.

**Core Dataset Interface:**

```python
from abc import ABC, abstractmethod
import pandas as pd
from typing import Optional, Dict, Any, List, Union
from pydantic import BaseModel, Field

class DatasetMetadata(BaseModel):
    """Metadata for dataset operations."""
    source_type: str = Field(description="Type of data source (database, file, s3, etc.)")
    format: Optional[str] = Field(description="Data format (csv, json, parquet, etc.)")
    schema: Optional[Dict[str, Any]] = Field(description="Data schema information")
    row_count: Optional[int] = Field(description="Total number of rows")
    column_count: Optional[int] = Field(description="Number of columns")
    size_bytes: Optional[int] = Field(description="Dataset size in bytes")
    last_modified: Optional[str] = Field(description="Last modification timestamp")
    version: Optional[str] = Field(description="Dataset version identifier")

class DatasetConfig(BaseModel):
    """Configuration for dataset access."""
    connection_params: Dict[str, Any] = Field(default_factory=dict, description="Connection parameters")
    query_params: Dict[str, Any] = Field(default_factory=dict, description="Query parameters")
    cache_enabled: bool = Field(default=True, description="Enable caching for performance")
    chunk_size: Optional[int] = Field(default=None, description="Chunk size for large datasets")

class BaseDataset(ABC):
    """Abstract base class for all dataset implementations."""

    def __init__(self, identifier: str, config: Optional[DatasetConfig] = None):
        self.identifier = identifier
        self.config = config or DatasetConfig()
        self._metadata: Optional[DatasetMetadata] = None
        self._cached_data: Optional[pd.DataFrame] = None

    @abstractmethod
    def load(self, **kwargs) -> pd.DataFrame:
        """Load data and return as pandas DataFrame."""
        pass

    @abstractmethod
    def get_metadata(self) -> DatasetMetadata:
        """Get dataset metadata."""
        pass

    def sample(self, n: int = 100, **kwargs) -> pd.DataFrame:
        """Get a sample of the dataset."""
        data = self.load(**kwargs)
        return data.sample(n=min(n, len(data)), random_state=42)

    def head(self, n: int = 5, **kwargs) -> pd.DataFrame:
        """Get first n rows of the dataset."""
        data = self.load(**kwargs)
        return data.head(n)

    def columns(self) -> List[str]:
        """Get column names."""
        metadata = self.get_metadata()
        if metadata.schema and 'columns' in metadata.schema:
            return metadata.schema['columns']
        # Fallback to loading a small sample
        sample_data = self.head(1)
        return sample_data.columns.tolist()

    def validate_schema(self, expected_columns: List[str]) -> bool:
        """Validate that dataset contains expected columns."""
        actual_columns = set(self.columns())
        expected_set = set(expected_columns)
        return expected_set.issubset(actual_columns)
```

**Dataset Implementations:**

The Dataset abstraction supports multiple data source types through specific implementations:

```python
class FileDataset(BaseDataset):
    """Dataset implementation for file-based data sources (CSV, JSON, Parquet, etc.)."""

    def __init__(self, file_path: str, config: Optional[DatasetConfig] = None):
        super().__init__(file_path, config)
        # Implementation handles file format detection and pandas loading

    def load(self, **kwargs) -> pd.DataFrame:
        # Loads file data with caching and format-specific parsing
        pass

    def get_metadata(self) -> DatasetMetadata:
        # Returns file metadata including size, modification time, schema
        pass

class DatabaseDataset(BaseDataset):
    """Dataset implementation for SQL database sources."""

    def __init__(self, connection_string: str, query: str, config: Optional[DatasetConfig] = None):
        super().__init__(f"db://{connection_string}", config)
        # Implementation handles SQLAlchemy engine management

    def load(self, **kwargs) -> pd.DataFrame:
        # Executes SQL query with chunked loading support
        pass

    def get_metadata(self) -> DatasetMetadata:
        # Returns query metadata including row count and column information
        pass

class S3Dataset(BaseDataset):
    """Dataset implementation for S3-based data sources."""

    def __init__(self, s3_uri: str, config: Optional[DatasetConfig] = None):
        super().__init__(s3_uri, config)
        # Implementation handles S3 client and URI parsing

    def load(self, **kwargs) -> pd.DataFrame:
        # Downloads and parses S3 objects with format detection
        pass

    def get_metadata(self) -> DatasetMetadata:
        # Returns S3 object metadata including size and modification time
        pass

class HuggingFaceDataset(BaseDataset):
    """Dataset implementation for HuggingFace datasets."""

    def __init__(self, dataset_name: str, config_name: Optional[str] = None,
                 split: str = "train", config: Optional[DatasetConfig] = None):
        super().__init__(f"hf://{dataset_name}/{config_name or 'default'}/{split}", config)
        # Implementation handles HuggingFace datasets library integration

    def load(self, **kwargs) -> pd.DataFrame:
        # Loads HuggingFace dataset and converts to pandas DataFrame
        pass

    def get_metadata(self) -> DatasetMetadata:
        # Returns HuggingFace dataset metadata and schema information
        pass
```

**Dataset Factory:**

```python
from typing import Type, Dict
from urllib.parse import urlparse

class DatasetFactory:
    """Factory for creating dataset instances based on identifier patterns."""

    _registry: Dict[str, Type[BaseDataset]] = {
        'file': FileDataset,
        'db': DatabaseDataset,
        'database': DatabaseDataset,
        's3': S3Dataset,
        'hf': HuggingFaceDataset,
        'huggingface': HuggingFaceDataset
    }

    @classmethod
    def create_dataset(cls, identifier: str, config: Optional[DatasetConfig] = None, **kwargs) -> BaseDataset:
        """Create dataset instance based on identifier pattern."""

        # Parse identifier to determine dataset type
        if identifier.startswith(('http://', 'https://')):
            # URL-based identifier
            parsed = urlparse(identifier)
            if 's3' in parsed.netloc:
                dataset_type = 's3'
            else:
                raise ValueError(f"Unsupported URL scheme: {identifier}")
        elif identifier.startswith('s3://'):
            dataset_type = 's3'
        elif identifier.startswith(('hf://', 'huggingface://')):
            dataset_type = 'hf'
        elif identifier.startswith(('db://', 'database://', 'postgresql://', 'mysql://', 'sqlite://')):
            dataset_type = 'db'
        elif os.path.exists(identifier) or '/' in identifier or '\\' in identifier:
            # File path
            dataset_type = 'file'
        else:
            # Default to HuggingFace for simple names
            dataset_type = 'hf'

        if dataset_type not in cls._registry:
            raise ValueError(f"Unsupported dataset type: {dataset_type}")

        dataset_class = cls._registry[dataset_type]

        # Handle special cases for different dataset types
        if dataset_type == 'hf' and not identifier.startswith(('hf://', 'huggingface://')):
            return dataset_class(identifier, config=config, **kwargs)
        elif dataset_type == 'db':
            # Extract query from kwargs for database datasets
            query = kwargs.pop('query', 'SELECT * FROM table')
            return dataset_class(identifier, query, config=config)
        else:
            return dataset_class(identifier, config=config, **kwargs)

    @classmethod
    def register_dataset_type(cls, name: str, dataset_class: Type[BaseDataset]):
        """Register a new dataset type."""
        cls._registry[name] = dataset_class
```

**Integration with Providers:**

```python
# Update existing provider models to use Dataset abstraction
class EvaluationRequest(TrustyAIRequest):
    """Request model for evaluation operations."""
    model: ModelReference
    tasks: List[str] = Field(description="Evaluation tasks to run")
    dataset: Optional[Union[DatasetReference, BaseDataset]] = None
    parameters: Dict[str, Any] = Field(default_factory=dict)

    class Config:
        schema_extra = {
            "examples": [
                {
                    "provider": "lm-evaluation-harness",
                    "model": {"identifier": "hf/gpt2", "type": "huggingface"},
                    "tasks": ["hellaswag", "arc_easy"],
                    "parameters": {"limit": 100, "batch_size": 8}
                },
                {
                    "provider": "ragas",
                    "model": {"identifier": "openai/gpt-4", "type": "api"},
                    "tasks": ["faithfulness", "answer_relevancy"],
                    "dataset": {"identifier": "s3://my-bucket/rag-data.parquet", "type": "s3", "format": "parquet"},
                    "parameters": {"embeddings_model": "openai/text-embedding-ada-002"}
                }
            ]
        }

# Provider usage example
class EvaluationProvider(BaseProvider):
    """Evaluation provider with Dataset support."""

    def evaluate(self, model: str, tasks: List[str], dataset: Optional[BaseDataset] = None, **kwargs):
        """Evaluate model on tasks with optional dataset."""

        if dataset is not None:
            # Load data from dataset abstraction
            evaluation_data = dataset.load()

            # Validate dataset has required columns for evaluation
            required_columns = self._get_required_columns(tasks)
            if not dataset.validate_schema(required_columns):
                raise ValueError(f"Dataset missing required columns: {required_columns}")

            # Use dataset in evaluation
            return self._run_evaluation_with_data(model, tasks, evaluation_data, **kwargs)
        else:
            # Use provider's default datasets
            return self._run_evaluation_default(model, tasks, **kwargs)
```

**Usage Examples:**

```python
from trustyai.datasets import DatasetFactory
from trustyai.providers import EvaluationProvider

# File-based dataset
file_dataset = DatasetFactory.create_dataset("data/evaluation_data.csv")

# Database dataset
db_dataset = DatasetFactory.create_dataset(
    "postgresql://user:pass@localhost/db",
    query="SELECT question, answer FROM qa_pairs WHERE category = 'science'"
)

# S3 dataset
s3_dataset = DatasetFactory.create_dataset("s3://my-bucket/evaluation-data.parquet")

# HuggingFace dataset
hf_dataset = DatasetFactory.create_dataset("squad", split="validation")

# Use with evaluation provider
eval_provider = EvaluationProvider(implementation="lm-evaluation-harness")

# Evaluate with custom dataset
results = eval_provider.evaluate(
    model="hf/microsoft/DialoGPT-medium",
    tasks=["question_answering"],
    dataset=file_dataset
)

# RAGAS evaluation with dataset
ragas_provider = EvaluationProvider(implementation="ragas")
rag_results = ragas_provider.evaluate(
    model="openai/gpt-4",
    tasks=["faithfulness", "answer_relevancy"],
    dataset=db_dataset  # Database with RAG evaluation data
)

# Dataset operations
print(f"Dataset columns: {file_dataset.columns()}")
print(f"Sample data:\n{file_dataset.sample(5)}")
print(f"Metadata: {file_dataset.get_metadata()}")
```

**CLI Integration with Datasets:**

The TrustyAI CLI also supports dataset operations for discovery, validation, and management:

```bash
# Dataset discovery and validation
trustyai dataset info "data/evaluation_data.csv"
trustyai dataset validate "s3://my-bucket/rag-data.parquet" --schema-file schema.json
trustyai dataset sample "postgresql://user:pass@localhost/db" --query "SELECT * FROM qa_pairs" --limit 10

# Dataset conversion and preprocessing
trustyai dataset convert "data/input.csv" --output "s3://my-bucket/converted.parquet" --format parquet
trustyai dataset preview "hf://squad" --split validation --limit 5

# Using datasets with evaluation
trustyai eval run \
    --provider lm-evaluation-harness \
    --model "hf/microsoft/DialoGPT-medium" \
    --tasks "question_answering" \
    --dataset "data/custom_qa.csv" \
    --dataset-columns "question,answer"

# Using datasets with RAGAS evaluation
trustyai eval run \
    --provider ragas \
    --model "openai/gpt-4" \
    --tasks "faithfulness,answer_relevancy" \
    --dataset "s3://my-bucket/rag-evaluation.parquet" \
    --parameters "embeddings_model=openai/text-embedding-ada-002"
```

This Dataset abstraction provides:

- **Universal Interface**: Consistent API across all data sources
- **Pandas Integration**: All datasets return pandas DataFrames
- **Flexible Sources**: Support for files, databases, S3, HuggingFace, and extensible for new sources
- **Metadata Support**: Rich metadata for dataset discovery and validation
- **Caching**: Built-in caching for performance improvement
- **Provider Integration**: Seamless integration with existing TrustyAI providers
- **Schema Validation**: Built-in validation for required columns
- **Factory Pattern**: Easy dataset creation based on identifier patterns

**Service Implementation Benefits:**

The Dataset abstraction significantly simplifies TrustyAI service metrics implementation by enabling runtime storage flexibility:

```python
# TrustyAI service specific metrics endpoints implementation
@app.post("/v2/metrics/spd/calculate")
async def calculate_spd(request: SPDRequest):
    """Calculate Statistical Parity Difference with runtime-specified data sources."""

    # Create dataset from runtime-specified storage
    dataset = DatasetFactory.create_dataset(
        request.dataset_uri,  # Could be "s3://bucket/data.parquet", "postgresql://...", etc.
        config=DatasetConfig(
            connection_params=request.connection_config,
            cache_enabled=True
        )
    )

    # Hardcoded provider for SPD metric - no provider parameter needed
    fairness_provider = FairnessProvider(implementation="fairlearn")

    # Provider handles data source abstraction internally
    spd_score = fairness_provider.statistical_parity_difference(
        dataset=dataset,  # Universal dataset interface
        protected_attribute=request.protected_attribute,
        target_column=request.target_column
    )

    return {"spd_score": spd_score}

@app.post("/v2/metrics/bleu/calculate")
async def calculate_bleu(request: BLEURequest):
    """Calculate BLEU score with runtime-specified data sources."""

    # Create dataset from runtime-specified storage
    dataset = DatasetFactory.create_dataset(
        request.dataset_uri,
        config=DatasetConfig(
            connection_params=request.connection_config,
            cache_enabled=True
        )
    )

    # Hardcoded provider for BLEU metric - no provider parameter needed
    eval_provider = EvaluationProvider(implementation="sacrebleu")

    # Provider handles data source abstraction internally
    bleu_score = eval_provider.bleu(
        dataset=dataset,  # Universal dataset interface
        reference_column=request.reference_column,
        candidate_column=request.candidate_column
    )

    return {"bleu_score": bleu_score}

@app.post("/v2/metrics/accuracy/calculate")
async def calculate_accuracy(request: AccuracyRequest):
    """Calculate accuracy with runtime-specified data sources."""

    # Create dataset from runtime-specified storage
    dataset = DatasetFactory.create_dataset(
        request.dataset_uri,
        config=DatasetConfig(
            connection_params=request.connection_config,
            cache_enabled=True
        )
    )

    # Hardcoded provider for accuracy metric - no provider parameter needed
    classification_provider = ClassificationProvider(implementation="sklearn")

    # Provider handles data source abstraction internally
    accuracy_score = classification_provider.accuracy(
        dataset=dataset,  # Universal dataset interface
        predictions_column=request.predictions_column,
        targets_column=request.targets_column
    )

    return {"accuracy": accuracy_score}
```

**Benefits for Ad-hoc Metrics Implementation:**

- **Storage Agnostic**: Service doesn't need to implement separate data connectors for each storage type
- **Runtime Configuration**: Data sources can be specified at request time without code changes
- **Simplified Code**: Metrics logic focuses on computation rather than data access patterns
- **Consistent Interface**: Same metrics code works with files, databases, S3, or any registered dataset type
- **Easy Testing**: Mock datasets can be easily substituted for unit testing
- **Performance Improvement**: Caching and chunking handled transparently by the dataset layer

**Example Service Usage Patterns:**

```bash
# Calculate SPD (Statistical Parity Difference) from database
POST /v2/metrics/spd/calculate
{
    "dataset_uri": "postgresql://prod-db/ml_data",
    "connection_config": {"query": "SELECT * FROM model_predictions WHERE date > '2024-01-01'"},
    "protected_attribute": "gender",
    "target_column": "approved"
}

# Calculate BLEU score from S3
POST /v2/metrics/bleu/calculate
{
    "dataset_uri": "s3://ml-datasets/translation-results.parquet",
    "reference_column": "reference_translation",
    "candidate_column": "model_translation"
}

# Calculate accuracy from HuggingFace dataset
POST /v2/metrics/accuracy/calculate
{
    "dataset_uri": "hf://my-org/validation-data",
    "connection_config": {"split": "test"},
    "predictions_column": "predicted_class",
    "targets_column": "true_class"
}
```

This approach eliminates the need for services to implement custom data connectors for each storage backend, allowing metrics implementations to focus purely on computation logic while the Dataset abstraction handles all data access complexity.

#### 6.4.1.2. Request/Response Models

**Base Request/Response Models:**

```python
class TrustyAIRequest(BaseModel):
    """Base request model for all provider operations."""
    provider: str = Field(description="Provider implementation to use")
    execution_mode: ExecutionMode = Field(default=ExecutionMode.LOCAL)
    configuration: Dict[str, Any] = Field(default_factory=dict)
    metadata: Optional[Dict[str, Any]] = Field(default_factory=dict)

class TrustyAIResponse(BaseModel):
    """Base response model for all provider operations."""
    metadata: TrustyAIMetadata
    status: JobStatus
    results: Optional[Dict[str, Any]] = None
    errors: List[str] = Field(default_factory=list)
    warnings: List[str] = Field(default_factory=list)
    execution_info: Dict[str, Any] = Field(default_factory=dict)

    def is_successful(self) -> bool:
        return self.status == JobStatus.COMPLETED and not self.errors
```

### 6.4.2. Common Provider API Arguments

##### 6.4.2.1. Evaluation Provider Scope

**Common Arguments Analysis:**

Based on analysis of LM-Evaluation-Harness and RAGAS implementations:

*LM-Evaluation-Harness:*
```bash
lm_eval --model hf --model_args pretrained=gpt2 --tasks hellaswag,arc_easy --batch_size 8 --limit 100 --device cuda --output_path results.json
```

*RAGAS:*
```bash
ragas evaluate --dataset dataset.json --metrics faithfulness,answer_relevancy,context_precision --output results.json --llm_model openai/gpt-4
```

**Standardised Common Arguments:**

```python
class EvaluationCommonArgs(BaseModel):
    """Common arguments across all evaluation providers."""
    # Required common arguments
    model_specification: str = Field(description="Model identifier/path")
    tasks: List[str] = Field(description="Evaluation tasks to run")

    # Optional common arguments with defaults
    batch_size: int = Field(default=1, description="Inference batch size")
    limit: Optional[int] = Field(default=None, description="Maximum samples to evaluate")
    output_path: Optional[str] = Field(default=None, description="Results output location")

    # Provider-specific extensions
    provider_args: Dict[str, Any] = Field(default_factory=dict, description="Provider-specific arguments")

# Provider-specific argument extensions
class LMEvalArgs(EvaluationCommonArgs):
    """LM-Evaluation-Harness specific arguments."""
    model_args: Optional[str] = Field(default=None, description="Model-specific arguments")
    device: Optional[str] = Field(default="auto", description="Execution device")
    num_fewshot: Optional[int] = Field(default=None, description="Number of few-shot examples")
    gen_kwargs: Optional[Dict[str, Any]] = Field(default_factory=dict)
    log_samples: bool = Field(default=False, description="Log individual samples")

class RAGASArgs(EvaluationCommonArgs):
    """RAGAS specific arguments."""
    dataset: str = Field(description="Dataset path for RAG evaluation")
    metrics: List[str] = Field(description="RAGAS metrics to compute (faithfulness, answer_relevancy, etc.)")
    llm_model: Optional[str] = Field(default=None, description="LLM model for evaluation")
    embeddings_model: Optional[str] = Field(default=None, description="Embeddings model for retrieval evaluation")
```

**Practical Usage Example:**

```python
from trustyai.providers import EvaluationProvider

# Common evaluation parameters
model_spec = "hf/microsoft/DialoGPT-medium"
tasks = ["hellaswag", "arc_easy"]
eval_params = {"batch_size": 8, "limit": 100}

# 1. Local execution - runs lm-eval directly on current machine
local_provider = EvaluationProvider(
    implementation="lm-evaluation-harness",
    execution_mode="local"
)

result_local = local_provider.evaluate(
    model=model_spec,
    tasks=tasks,
    parameters=eval_params
)

# 2. Kubernetes execution - deploys evaluation job to cluster
k8s_provider = EvaluationProvider(
    implementation="lm-evaluation-harness",
    execution_mode="kubernetes",
    config={
        "namespace": "trustyai-eval",
        "cpu": "4",
        "memory": "8Gi",
        "timeout": 3600
    }
)

result_k8s = k8s_provider.evaluate(
    model=model_spec,
    tasks=tasks,
    parameters=eval_params
)

# All results have identical structure regardless of execution mode
print(f"Local accuracy: {result_local.accuracy}")
print(f"K8s accuracy: {result_k8s.accuracy}")

# External API calls can be configured within local execution
api_provider = EvaluationProvider(
    implementation="lm-evaluation-harness",
    execution_mode="local",
    config={
        "api_endpoint": "https://eval-api.trustyai.com",
        "api_key": "your-api-key"
    }
)

result_api = api_provider.evaluate(
    model=model_spec,
    tasks=tasks,
    parameters=eval_params
)

print(f"API accuracy: {result_api.accuracy}")

# Different provider (RAGAS for RAG evaluation)
ragas_provider = EvaluationProvider(
    implementation="ragas",
    execution_mode="local"
)

rag_result = ragas_provider.evaluate(
    model="openai/gpt-4",
    tasks=["faithfulness", "answer_relevancy", "context_precision"],
    parameters={
        "dataset": "rag_dataset.json",
        "embeddings_model": "openai/text-embedding-ada-002"
    }
)

print(f"RAG faithfulness: {rag_result.faithfulness}")
print(f"RAG answer relevancy: {rag_result.answer_relevancy}")
print(f"RAG context precision: {rag_result.context_precision}")
```

##### 6.4.2.2. Red Teaming Provider Scope

**Common Arguments Analysis:**

Based on analysis of Garak and PurpleLlama CyberSecEval implementations:

*Garak:*
```bash
garak --model_type huggingface --model_name microsoft/DialoGPT-medium --probes encoding.InjectBase64 --detectors base.TriggerListDetector --report_prefix results
```

*PurpleLlama CyberSecEval:*
```bash
python3 -m CybersecurityBenchmarks.benchmark.run --benchmark=autocomplete --prompt-path prompts.json --response-path responses.json --llm-under-test=meta-llama/Llama-2-7b-chat-hf
```

**Standardised Common Arguments:**

```python
class RedTeamingCommonArgs(BaseModel):
    """Common arguments across all red teaming providers."""
    # Required common arguments
    model_specification: str = Field(description="Model identifier/path")
    benchmark_type: str = Field(description="Type of red teaming benchmark")

    # Optional common arguments
    input_data: Optional[str] = Field(default=None, description="Input prompts/dataset path")
    output_path: Optional[str] = Field(default=None, description="Results output location")
    max_queries: Optional[int] = Field(default=None, description="Maximum queries to execute")

    # Provider-specific extensions
    provider_args: Dict[str, Any] = Field(default_factory=dict)

# Provider-specific argument extensions
class GarakArgs(RedTeamingCommonArgs):
    """Garak specific arguments."""
    model_type: str = Field(description="Model type (huggingface, openai, etc.)")
    probes: List[str] = Field(description="Probe modules to run")
    detectors: List[str] = Field(description="Detector modules to use")
    report_prefix: Optional[str] = Field(default="garak_report", description="Report file prefix")
    generations: int = Field(default=10, description="Generations per probe")

class CyberSecEvalArgs(RedTeamingCommonArgs):
    """PurpleLlama CyberSecEval specific arguments."""
    prompt_path: str = Field(description="Path to prompt dataset")
    response_path: str = Field(description="Path to store responses")
    llm_under_test: str = Field(description="Model specification for testing")
    judge_response_path: Optional[str] = Field(default=None, description="Judge evaluation output")
```

##### 6.4.2.3. Unified Provider Configuration

**Common Configuration Schema:**

```python
class ProviderCommonConfig(BaseModel):
    """Unified configuration schema for all providers."""
    # Core identification
    provider_type: str = Field(description="Provider scope (evaluation, red_teaming, explainability, bias_detection)")
    implementation: str = Field(description="Specific implementation name")

    # Execution configuration
    execution_mode: ExecutionMode = Field(default=ExecutionMode.LOCAL)
    timeout: int = Field(default=3600, description="Operation timeout in seconds")

    # Common arguments (scope-dependent)
    common_args: Union[EvaluationCommonArgs, RedTeamingCommonArgs] = Field(description="Scope-specific common arguments")

    # Backend-specific extensions
    backend_config: Dict[str, Any] = Field(default_factory=dict, description="Backend-specific configuration")

    # Resource configuration
    resource_limits: Optional[Dict[str, str]] = Field(default=None, description="Resource limits for execution")

class BackendExtension(BaseModel):
    """Base class for backend-specific extensions."""
    implementation: str
    config: Dict[str, Any]

    class Config:
        extra = "allow"  # Allow additional fields for backend flexibility

# Example backend extensions
class KubernetesBackendConfig(BackendExtension):
    """Kubernetes-specific backend configuration."""
    namespace: str
    image: Optional[str] = None
    service_account: Optional[str] = None
    node_selector: Dict[str, str] = Field(default_factory=dict)

class LocalBackendConfig(BackendExtension):
    """Local execution backend configuration."""
    working_directory: Optional[str] = None
    environment_variables: Dict[str, str] = Field(default_factory=dict)
    virtual_environment: Optional[str] = None
```

**Usage Example:**

```python
# Evaluation provider configuration
eval_config = ProviderCommonConfig(
    provider_type="evaluation",
    implementation="lm-evaluation-harness",
    common_args=LMEvalArgs(
        model_specification="hf/gpt2",
        tasks=["hellaswag", "arc_easy"],
        batch_size=8,
        limit=100,
        model_args="pretrained=gpt2",
        device="cuda"
    ),
    backend_config={
        "kubernetes": KubernetesBackendConfig(
            implementation="k8s-job",
            namespace="trustyai-eval",
            image="trustyai/lm-eval:latest",
            config={"cpu": "4", "memory": "8Gi"}
        )
    }
)

# Red teaming provider configuration
redteam_config = ProviderCommonConfig(
    provider_type="red_teaming",
    implementation="garak",
    common_args=GarakArgs(
        model_specification="microsoft/DialoGPT-medium",
        benchmark_type="encoding_attacks",
        model_type="huggingface",
        probes=["encoding.InjectBase64"],
        detectors=["base.TriggerListDetector"],
        generations=20
    ),
    backend_config={
        "local": LocalBackendConfig(
            implementation="local-process",
            working_directory="/tmp/garak_workspace",
            config={"python_env": "garak_env"}
        )
    }
)
```

This architecture provides:

- **Standardisation**: Common arguments across providers within each scope
- **Flexibility**: Provider-specific extensions through inheritance
- **Backend Agnostic**: Separate backend configuration from provider logic
- **Type Safety**: Full validation for all configuration levels
- **Extensibility**: Easy inclusion of new providers and backend types

#### 6.4.3. Provider-Specific Schemas

**Evaluation Provider Models:**

```python
class EvaluationRequest(TrustyAIRequest):
    """Request model for evaluation operations."""
    model: ModelReference
    tasks: List[str] = Field(description="Evaluation tasks to run")
    dataset: Optional[DatasetReference] = None
    parameters: Dict[str, Any] = Field(default_factory=dict)

    class Config:
        schema_extra = {
            "example": {
                "provider": "lm-evaluation-harness",
                "model": {"identifier": "hf/gpt2", "type": "huggingface"},
                "tasks": ["hellaswag", "arc_easy"],
                "parameters": {"limit": 100, "batch_size": 8}
            }
        }

class EvaluationMetric(BaseModel):
    """Individual evaluation metric result."""
    name: str = Field(description="Metric name")
    value: Union[float, int, str] = Field(description="Metric value")
    unit: Optional[str] = Field(description="Metric unit if applicable")
    confidence_interval: Optional[Dict[str, float]] = None
    metadata: Dict[str, Any] = Field(default_factory=dict)

class TaskResult(BaseModel):
    """Results for a single evaluation task."""
    task_name: str
    metrics: List[EvaluationMetric]
    samples_evaluated: int
    execution_time: float = Field(description="Execution time in seconds")
    raw_results: Optional[Dict[str, Any]] = Field(description="Provider-specific raw results")

class EvaluationResponse(TrustyAIResponse):
    """Response model for evaluation operations."""
    model: ModelReference
    tasks: List[TaskResult] = Field(default_factory=list)
    summary_metrics: List[EvaluationMetric] = Field(default_factory=list)
    total_samples: int = 0
    total_execution_time: float = 0.0
```

**Explainability Provider Models:**

```python
class ExplainabilityRequest(TrustyAIRequest):
    """Request model for explainability operations."""
    model: ModelReference
    input_data: Dict[str, Any] = Field(description="Input data to explain")
    explanation_type: str = Field(description="Type of explanation (feature_importance, counterfactual, etc.)")
    parameters: Dict[str, Any] = Field(default_factory=dict)

class FeatureImportance(BaseModel):
    """Feature importance explanation."""
    feature_name: str
    importance_score: float
    confidence: Optional[float] = None
    explanation: Optional[str] = None

class ExplanationResult(BaseModel):
    """Explanation result."""
    explanation_type: str
    prediction: Optional[Any] = None
    confidence: Optional[float] = None
    feature_importances: List[FeatureImportance] = Field(default_factory=list)
    counterfactuals: Optional[List[Dict[str, Any]]] = None
    visualisation_data: Optional[Dict[str, Any]] = None
    raw_explanation: Optional[Dict[str, Any]] = None

class ExplainabilityResponse(TrustyAIResponse):
    """Response model for explainability operations."""
    model: ModelReference
    input_data: Dict[str, Any]
    explanations: List[ExplanationResult] = Field(default_factory=list)
```

**Bias Detection Provider Models:**

```python
class BiasDetectionRequest(TrustyAIRequest):
    """Request model for bias detection operations."""
    model: ModelReference
    dataset: DatasetReference
    protected_attributes: List[str] = Field(description="Attributes to check for bias")
    fairness_metrics: List[str] = Field(description="Fairness metrics to calculate")
    parameters: Dict[str, Any] = Field(default_factory=dict)

class BiasMetric(BaseModel):
    """Individual bias metric result."""
    metric_name: str
    protected_attribute: str
    value: float
    threshold: Optional[float] = None
    is_fair: Optional[bool] = None
    interpretation: Optional[str] = None

class BiasDetectionResponse(TrustyAIResponse):
    """Response model for bias detection operations."""
    model: ModelReference
    dataset: DatasetReference
    protected_attributes: List[str]
    bias_metrics: List[BiasMetric] = Field(default_factory=list)
    overall_fairness_score: Optional[float] = None
    recommendations: List[str] = Field(default_factory=list)
```

#### 6.4.4. Validation and Serialisation

**Validation Patterns:**

```python
from pydantic import validator, root_validator

class EvaluationRequest(TrustyAIRequest):
    # ... fields ...

    @validator('tasks')
    def validate_tasks(cls, v):
        if not v:
            raise ValueError('At least one task must be specified')
        return v

    @validator('model')
    def validate_model_reference(cls, v):
        if v.type == 'huggingface' and not v.identifier.startswith('hf/'):
            raise ValueError('HuggingFace models must start with "hf/"')
        return v

    @root_validator
    def validate_execution_mode_params(cls, values):
        execution_mode = values.get('execution_mode')
        config = values.get('configuration', {})

        if execution_mode == ExecutionMode.KUBERNETES:
            required_k8s_params = ['namespace']
            missing = [p for p in required_k8s_params if p not in config]
            if missing:
                raise ValueError(f'Kubernetes execution requires: {missing}')

        return values
```

**Serialisation Support:**

```python
class TrustyAIEncoder(json.JSONEncoder):
    """Custom JSON encoder for TrustyAI data models."""

    def default(self, obj):
        if isinstance(obj, BaseModel):
            return obj.dict()
        elif isinstance(obj, datetime):
            return obj.isoformat()
        elif isinstance(obj, Enum):
            return obj.value
        return super().default(obj)

# Usage
response = EvaluationResponse(...)
json_str = json.dumps(response, cls=TrustyAIEncoder, indent=2)

# Deserialisation
response_dict = json.loads(json_str)
response_obj = EvaluationResponse(**response_dict)
```

#### 6.4.5. Configuration Schema

**Provider Configuration Models:**

```python
class ProviderConfig(BaseModel):
    """Base configuration for all providers."""
    implementation: str
    execution_mode: ExecutionMode = ExecutionMode.LOCAL
    timeout: int = Field(default=3600, description="Operation timeout in seconds")
    retry_policy: Dict[str, Any] = Field(default_factory=lambda: {
        "max_attempts": 3,
        "backoff_factor": 2.0,
        "exceptions": ["ConnectionError", "TimeoutError"]
    })

class LocalExecutionConfig(BaseModel):
    """Configuration for local execution."""
    working_directory: Optional[str] = None
    environment_variables: Dict[str, str] = Field(default_factory=dict)
    resource_limits: Optional[Dict[str, str]] = None

class KubernetesExecutionConfig(BaseModel):
    """Configuration for Kubernetes execution."""
    namespace: str
    job_template: Optional[str] = None
    resource_limits: Dict[str, str] = Field(default_factory=lambda: {
        "cpu": "2",
        "memory": "4Gi"
    })
    service_account: Optional[str] = None
    node_selector: Dict[str, str] = Field(default_factory=dict)
    tolerations: List[Dict[str, Any]] = Field(default_factory=list)
```

#### 6.4.6. Error Handling Schema

**Standardised Error Models:**

```python
class TrustyAIError(BaseModel):
    """Standardised error representation."""
    code: str = Field(description="Error code for programmatic handling")
    message: str = Field(description="Human-readable error message")
    category: str = Field(description="Error category (validation, execution, system)")
    context: Dict[str, Any] = Field(default_factory=dict)
    suggestion: Optional[str] = Field(description="Suggested resolution")

class ValidationError(TrustyAIError):
    """Validation-specific error."""
    field: Optional[str] = Field(description="Field that failed validation")
    invalid_value: Optional[Any] = Field(description="The invalid value")

class ExecutionError(TrustyAIError):
    """Execution-specific error."""
    exit_code: Optional[int] = None
    stdout: Optional[str] = None
    stderr: Optional[str] = None
```

This complete data model architecture ensures:

- **Type Safety**: All data is validated at runtime
- **Consistency**: Common schemas across all providers and operations
- **Extensibility**: Base models can be extended for provider-specific needs
- **Interoperability**: Standard serialisation formats for API communication
- **Debugging**: Error models with user-friendly suggestions
- **Documentation**: Self-documenting schemas with examples and descriptions

### 6.5. CLI Interface

The TrustyAI CLI provides a consistent interface across all deployment targets, with minimal syntax differences between local and Kubernetes execution. This design ensures users can easily transition between development (local) and production (Kubernetes) environments.

#### 6.5.1. Consistent Command Structure

The CLI follows a consistent pattern: `trustyai <scope> execute --provider <provider> --execution-mode <mode> [options]`

**Local Execution:**
```bash
# Local evaluation execution
trustyai eval execute --provider lm-evaluation-harness --execution-mode local --model "hf/microsoft/DialoGPT-medium" --limit 10 --tasks "hellaswag,arc"
```

**Kubernetes Execution:**
```bash
# Kubernetes evaluation execution
trustyai eval execute --provider lm-evaluation-harness --execution-mode kubernetes --model "hf/microsoft/DialoGPT-medium" --limit 10 --tasks "hellaswag,arc" --namespace trustyai-eval
```

#### 6.5.2. Command Comparison

| Aspect | Local | Kubernetes |
|--------|-------|------------|
| **Base command** | `trustyai eval execute --execution-mode local` | `trustyai eval execute --execution-mode kubernetes` |
| **Core arguments** | `--provider`, `--model`, `--limit` | `--provider`, `--model`, `--limit` (identical) |
| **Execution-specific** | Local paths, environment variables | `--namespace`, `--cpu`, `--memory`, `--image` |
| **Validation** | Local validator (binary, deps) | K8s validator (cluster, operator, namespace) |
| **Output** | TrustyAI evaluation result object | TrustyAI evaluation result object (identical) |

#### 6.5.3. Advanced CLI Features

**Configuration Profiles:**
```bash
# Save common configurations
trustyai config set-profile dev --provider lm-evaluation-harness --model "hf/gpt2" --limit 5
trustyai config set-profile prod --provider lm-evaluation-harness --execution-mode kubernetes --namespace trustyai-prod --cpu 8 --memory 16Gi

# Use profiles
trustyai eval execute --profile dev --tasks "custom_task"
trustyai eval execute --profile prod --tasks "custom_task"
```

**Interactive Mode:**
```bash
# Interactive configuration wizard
trustyai eval execute --interactive
# Prompts for provider, model, tasks, etc.

# Watch mode for Kubernetes deployments
trustyai eval execute --watch --provider lm-evaluation-harness --execution-mode kubernetes --model "hf/model" --namespace test
# Streams job logs and status updates
```

**Validation-First Approach:**
```bash
# Dry-run with validation only
trustyai eval execute --dry-run --provider lm-evaluation-harness --execution-mode local --model "hf/model"
trustyai eval execute --dry-run --provider lm-evaluation-harness --execution-mode kubernetes --namespace test

# Force execution despite warnings
trustyai eval execute --force --provider lm-evaluation-harness --execution-mode local --model "hf/model"
```

This CLI design ensures that users learn one interface that works consistently across all deployment scenarios, reducing cognitive load and enabling smooth transitions between different environments and target platforms.

### 6.6. Provider Architecture

Providers in the TrustyAI SDK offer universal APIs that abstract the underlying implementation details, enabling seamless switching between different execution modes.

#### 6.6.1. Provider Interface Design

Each provider type defines a universal API that remains consistent regardless of the underlying implementation or execution environment. This allows the same code to work with different backends without modification.

#### 6.6.2. Example: Evaluation Provider

The `EvaluationProvider` shows how the provider architecture enables more flexible patterns:

**Universal API:**
```python
from trustyai.providers import EvaluationProvider

# Same API regardless of implementation
eval_provider = EvaluationProvider(implementation="lm-evaluation-harness")
result = eval_provider.evaluate(model, tasks, parameters)
```

**Implementation Options:**

1. **Local Implementation:**
   - Provider transforms parameters into direct `lm-eval` CLI invocations
   - Executes evaluation locally on the current machine
   - Returns results directly from the local process

2. **Kubernetes Implementation:**
   - Provider builds a Custom Resource (CR) from the input parameters
   - Deploys the evaluation job on a Kubernetes cluster
   - Monitors job completion and retrieves results
   - Provides the same response format as local execution

**Implementation Switching:**
```python
# Local execution
local_provider = EvaluationProvider(
    implementation="lm-evaluation-harness",
    execution_mode="local"
)

# Kubernetes execution
# Note: Cluster configuration is automatically detected from environment
k8s_provider = EvaluationProvider(
    implementation="lm-evaluation-harness",
    execution_mode="kubernetes"
)

# Same API, different execution environments
result1 = local_provider.evaluate(params)
result2 = k8s_provider.evaluate(params)  # Identical interface
```

**Configuration Examples:**

```python
# Local configuration
local_config = {
    "implementation": "lm-evaluation-harness",
    "execution_mode": "local"
}

# Kubernetes configuration
k8s_config = {
    "implementation": "lm-evaluation-harness",
    "execution_mode": "kubernetes",
    "namespace": "trustyai-eval",
    "job_template": "lm-eval-job-template",
    "resource_limits": {"cpu": "4", "memory": "8Gi"}
}
```

This architecture enables:
- **Unified Developer Experience**: Same API regardless of deployment target
- **Flexible Deployment**: Switch between different execution modes
- **Implementation Abstraction**: Add new evaluation frameworks without changing client code
- **Resource Management**: Route intensive evaluations to appropriate compute resources
- **Implementation Shield**: Users are protected from implementation-specific idiosyncrasies (e.g., Kubernetes resource management, networking configurations, or vendor-specific API quirks) as providers handle all necessary preparations and adaptations internally

**Workflow Diagram:**

```mermaid
graph TD
    A["Client calls eval_provider.evaluate()"] --> B{Execution Mode?}

    B -->|Local| C[Run Local LM-Eval Validator]
    B -->|Kubernetes| D[Run K8s LM-Eval Validator]

    C --> C1{Validation Passed?}
    D --> D1{Validation Passed?}

    C1 -->|No| C2[Return validation errors with suggestions]
    D1 -->|No| D2[Return validation errors with suggestions]

    C1 -->|Yes| E[Transform parameters to lm-eval CLI args]
    D1 -->|Yes| F[Build Custom Resource from parameters]

    E --> G[Execute lm-eval locally]
    G --> H[Capture stdout/stderr]
    H --> I[Parse results]

    F --> J[Deploy CR to K8s cluster]
    J --> K[Monitor job status]
    K --> L{Job Complete?}
    L -->|No| K
    L -->|Yes| M[Retrieve job logs/results]
    M --> N[Parse results from cluster]

    I --> O[Return standardised result format]
    N --> O
    C2 --> P[Client receives validation feedback]
    D2 --> P
    O --> Q[Client receives results]

    subgraph "Local Execution Flow"
        C
        C1
        C2
        E
        G
        H
        I
    end

    subgraph "Kubernetes Execution Flow"
        D
        D1
        D2
        F
        J
        K
        L
        M
        N
    end

    subgraph "Universal Interface"
        A
        B
        O
        P
        Q
    end
```

#### 6.6.3. Validators

Validators are classes that check if the system is ready before running operations. They provide helpful error messages instead of letting users see confusing technical errors.

**Validator Architecture:**

Each implementation has validators that check what's needed:

```python
from trustyai.validators import BaseValidator

class K8sLMEvalValidator(BaseValidator):
    def validate(self) -> ValidationResult:
        checks = [
            self._check_cluster_connectivity(),
            self._check_trustyai_operator(),
            self._check_namespace_exists(),
            self._check_resource_quotas()
        ]
        return ValidationResult(checks)

    def _check_cluster_connectivity(self) -> CheckResult:
        # Check if we can connect to the cluster
        try:
            # Test kubectl/cluster access
            return CheckResult.success("Cluster connection working")
        except Exception as e:
            return CheckResult.error(
                "Cannot connect to Kubernetes cluster",
                suggestion="Check your kubeconfig file and cluster status"
            )

    def _check_trustyai_operator(self) -> CheckResult:
        # Check if TrustyAI operator is installed
        try:
            # Look for TrustyAI operator in cluster
            return CheckResult.success("TrustyAI operator found")
        except Exception as e:
            return CheckResult.error(
                "TrustyAI operator not found in cluster",
                suggestion="Install TrustyAI operator: kubectl apply -f trustyai-operator.yaml"
            )

    def _check_namespace_exists(self) -> CheckResult:
        # Check if target namespace exists
        pass

    def _check_resource_quotas(self) -> CheckResult:
        # Check if there are enough resources available
        pass
```

**Implementation-Specific Validators:**

1. **Local LM-Eval Validator:**
   - Check Python environment and required packages
   - Check if model can be accessed (local paths, HuggingFace cache)
   - Check if there's enough disk space and memory

2. **Kubernetes LM-Eval Validator:**
   - Check if we can connect to the cluster
   - Check if TrustyAI operator is installed and working
   - Check if target namespace exists with proper permissions
   - Check if resource limits allow job execution
   - Check if required ConfigMaps and Secrets are present

**Validation Integration:**

Validators run automatically before execution or can be called directly:

```python
# Automatic validation before execution
eval_provider = EvaluationProvider(implementation="lm-evaluation-harness", execution_mode="kubernetes")
result = eval_provider.evaluate(params)  # Validates first, then executes

# Manual validation for troubleshooting
validation_result = eval_provider.validate()
if not validation_result.is_valid:
    for issue in validation_result.issues:
        print(f"❌ {issue.category}: {issue.message}")
        if issue.suggestion:
            print(f"   💡 Suggestion: {issue.suggestion}")
else:
    print("✅ All system checks passed")
```

**Example Validation Output:**

```
❌ Cluster Connectivity: Unable to connect to Kubernetes cluster
   💡 Suggestion: Check your kubeconfig file and cluster status

❌ TrustyAI Operator: TrustyAI operator not found in cluster
   💡 Suggestion: Install TrustyAI operator: kubectl apply -f trustyai-operator.yaml

❌ Namespace: Namespace 'trustyai-eval' does not exist
   💡 Suggestion: Create namespace: kubectl create namespace trustyai-eval

✅ Resource Quotas: Sufficient resources available
```

This validation system helps users get clear, helpful feedback about what needs to be fixed, making debugging much easier.

These validators support the requirement that each provider must work with at least one execution mode (Local or Kubernetes). The validator framework helps providers check if their target platform is ready to run, whether they use LocalExecutor, KubernetesExecutor, or both.

#### 6.6.4. Example: Red Team Provider *(Not Implemented)*

The same provider architecture pattern could extend to other AI safety domains. For instance, a `RedTeamProvider` for security testing would show the flexibility of the universal API approach:

**Universal API:**
```python
from trustyai.providers import RedTeamProvider

# Same API regardless of implementation
redteam_provider = RedTeamProvider(implementation="garak")
result = redteam_provider.test(
    model="model_name",
    attacks=["injection", "jailbreak"],
    dataset="safety_prompts.json"
)
```

**Implementation Options:**

1. **Local Implementation:**
   - Provider transforms parameters into direct `garak` CLI invocations
   - Executes security testing locally on the current machine
   - Returns vulnerability assessment results directly

2. **Kubernetes Implementation:**
   - Provider builds a Custom Resource (CR) for red team testing jobs
   - Deploys complete security testing on a Kubernetes cluster
   - Monitors job completion and aggregates security findings
   - Provides the same response format as local execution

**CLI Usage Example:**
```bash
# Local red team testing
trustyai redteam run --provider garak --model "local_llm" --attacks "injection,jailbreak" --dataset "safety_prompts.json"

# Kubernetes deployment for complete testing
trustyai redteam deploy --provider garak --model "api://production-model" --attacks "adversarial_suite" --namespace security --timeout 24h
```

This shows how the provider architecture naturally extends across different AI safety domains, maintaining consistency while supporting domain-specific requirements.

### 6.7. Usage Patterns

The TrustyAI SDK enables multiple usage patterns across different target environments:

**CLI Usage:**
```bash
# Local execution using 'run' command
trustyai eval run --provider lm-evaluation-harness --model "hf/microsoft/DialoGPT-medium" --tasks "hellaswag,arc" --limit 10

# Kubernetes execution using 'deploy' command
trustyai eval deploy --provider lm-evaluation-harness --model "hf/microsoft/DialoGPT-medium" --tasks "hellaswag,arc" --limit 10 --namespace trustyai-eval

# Validation-only mode
trustyai eval run --dry-run --provider lm-evaluation-harness --model "hf/model"
```

**REST Service Usage:**
```bash
# Deploy TrustyAI service locally
docker-compose up trustyai-service

# Or deploy to Kubernetes
kubectl apply -f kubernetes/deployment.yaml

# Use REST endpoints for evaluation
curl -X POST "http://localhost:8000/api/v1/evaluation/run" \
  -H "Content-Type: application/json" \
  -d '{
    "provider": "lm-evaluation-harness",
    "model": "hf/microsoft/DialoGPT-medium",
    "tasks": ["hellaswag", "arc"],
    "execution_mode": "kubernetes"
  }'

# Monitor job status
curl "http://localhost:8000/api/v1/evaluation/jobs/{job_id}"

# Use explainability endpoints
curl -X POST "http://localhost:8000/api/v1/explainability/explain" \
  -H "Content-Type: application/json" \
  -d '{
    "provider": "shap",
    "model": "trained_model.pkl",
    "input_data": {"features": [1, 2, 3, 4]}
  }'
```

**TrustyAI SDK Library Usage:**
```python
# Direct usage in applications
import trustyai
from trustyai.providers import EvaluationProvider, ExplainabilityProvider

# Local evaluation
eval_provider = EvaluationProvider(implementation="lm-evaluation-harness")
results = eval_provider.evaluate(
    model="hf/microsoft/DialoGPT-medium",
    tasks=["hellaswag", "arc"],
    parameters={"limit": 10}
)

# Kubernetes evaluation (same API)
k8s_eval_provider = EvaluationProvider(
    implementation="lm-evaluation-harness",
    execution_mode="kubernetes"
)
results = k8s_eval_provider.evaluate(
    model="hf/microsoft/DialoGPT-medium",
    tasks=["hellaswag", "arc"],
    parameters={"limit": 100}
)
```

**Service Implementation (FastAPI consuming trustyai):**
```python
# Services import and use the SDK
from fastapi import FastAPI
import trustyai
from trustyai.providers import EvaluationProvider

app = FastAPI()

@app.post("/custom-eval")
async def custom_evaluation(request: EvalRequest):
    # Service uses SDK internally
    provider = EvaluationProvider(implementation=request.provider)
    result = provider.evaluate(
        model=request.model,
        tasks=request.tasks
    )
    return {"evaluation_result": result}
```

**Jupyter Notebook Usage:**
```python
# Interactive data science workflows
import trustyai
import pandas as pd

# Load data and configure provider in notebook environment
data = pd.read_csv('dataset.csv')
eval_provider = trustyai.providers.EvaluationProvider(implementation="lm-evaluation-harness")

# Interactive evaluation with rich outputs
results = eval_provider.evaluate(model, data)
results.plot()  # Built-in visualisation support

# Explain model decisions interactively
explainer = trustyai.providers.ExplainabilityProvider(implementation="shap")
explanation = explainer.explain(model, sample_data)
explanation.visualise()
```

**SDK and Client Library Usage:**
```python
# Python applications using trustyai directly
import trustyai

# Other languages communicate with REST service
# JavaScript/TypeScript example:
# const response = await fetch('/api/v1/evaluation/run', {
#   method: 'POST',
#   body: JSON.stringify({
#     provider: 'lm-evaluation-harness',
#     model: 'hf/microsoft/DialoGPT-medium',
#     tasks: ['hellaswag', 'arc']
#   })
# });
```

This architecture provides flexibility across deployment environments while maintaining consistent interfaces and allowing seamless transitions between local development and production scenarios.

### 6.8. TrustyAI Service

The TrustyAI Service is a REST API that uses the TrustyAI SDK as its foundation. This service will replace the existing Java-based TrustyAI service, providing the same capabilities through a Python implementation that can leverage the broader Python AI ecosystem.

#### 6.8.1. Component Interaction Flow

```mermaid
sequenceDiagram
    participant Client
    participant Service as TrustyAI Service (Python)
    participant Core as TrustyAI SDK
    participant Provider as Provider Interface
    participant Impl as Provider Implementation

    Note over Service: Service imports trustyai package
    Client->>Service: HTTP Request
    Service->>Core: Call SDK methods
    Core->>Provider: Get appropriate provider
    Provider->>Impl: Execute implementation
    Impl->>Provider: Return result
    Provider->>Core: Process result
    Core->>Service: Return processed result
    Service->>Client: HTTP Response
```

**Service Architecture:**

1. **TrustyAI REST Service (Python)**:
   - Imports and uses the `trustyai` SDK
   - Provides HTTP endpoints for all provider capabilities
   - Handles authentication, authorisation, and API documentation
   - Replaces the existing Java-based TrustyAI service
   - Maintains compatibility with existing integrations (ModelMesh, ODH/RHODS)

2. **Migration Path**:
   - The Python service provides the same external API as the Java service
   - Existing clients can migrate without code changes
   - New features will be added to the Python service only
   - The Java service will be deprecated after migration is complete

## 7. Threat model

TBD

## 8. Alternatives Considered / Rejected

- **Maintaining separate APIs with shared documentation**: Would not solve the underlying architectural fragmentation.
- **Creating adapter layers for each component**: Introduces additional complexity without standardising the core interfaces.
- **Using a event-driven architecture**: Would add unnecessary complexity for synchronous operations that form the majority of use cases.
- **Monolithic API design**: Would limit flexibility and make component isolation more difficult.

## 9. Challenges

- Having to support diverse functionality across AI safety domains
- Ensuring appropriate abstractions that don't leak implementation details
- Designing provider interfaces that support various implementation approaches

## 10. Dependencies

- Creation and packaging of the `trustyai` SDK for PyPI distribution
  - Implementation of metrics providers for evaluation, explainability, and bias detection
  - Provider interfaces with local and Kubernetes execution modes
  - Validation frameworks for system checks
  - Common data models and error handling patterns
  - **Platform-specific provider implementations**: Each provider must implement at least one of LocalExecutor or KubernetesExecutor, with both implementations recommended for maximum deployment flexibility

- **Python REST Service Development:**
  - Service implementation that imports and uses the `trustyai` SDK
  - HTTP endpoints for all provider capabilities (evaluation, explainability, guardrails)
  - Kubernetes deployment configurations using TrustyAI operator (when available)
  - Health monitoring and observability endpoints


- **Kubernetes Integration:**
  - TrustyAI operator integration for Kubernetes deployments
  - Custom Resource Definitions for evaluation jobs and other operations
  - Namespace and RBAC configuration templates
  - Resource management and scaling policies

- Documentation for both SDK usage and service implementation patterns
- Updates to documentation and developer guides
- Potential changes to client libraries and integrations
- Updates to testing frameworks for both SDK and services
- Creation of reference implementations for each provider type
- CI/CD pipelines for SDK releases and service deployments

## 11. Consequences if not completed

- Continued fragmentation of TrustyAI interfaces
- Increased developer learning curve
- Hard to maintain and extend the system
- Reduced ability to provide complete AI safety capabilities
- Continued dependency on Java ecosystem limiting community contributions
- Difficulty integrating with Python-based AI/ML tools and libraries
