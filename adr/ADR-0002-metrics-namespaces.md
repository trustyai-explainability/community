---
num: 2 # allocate an id when the draft is created
title: ADR template
status: "Draft" # One of Draft, Accepted, Rejected
authors:
  - "ruivieira" # One item for each author, as github id or "firstname lastname"
tags:
  - "core,python,service" # e.g. service, python, java, etc
---

## Title

    Metrics namespaces

## Context and Problem Statement

Currently, the metrics namespace in `explainability-core` consist of a single package (`org.kie.trustyai.explainability.metrics`) with a single class (`FairnessMetrics`).
As more metrics will be added in the future, it is important to have a clear structure for the metrics namespace. The namespace should be easy to navigate and understand and reflect a general taxonomy of fairness metrics.
This will also allow extracting common functionality into abstract classes, and make it easier to add new metrics.
Additional metrics may also be provided by third-party libraries, and it is important to have a clear separation between the metrics provided by `explainability-core` and those provided by third-party libraries, as well as accomodating potential name clashes and conflicts.

## Goals

Refractor `explainability-core` to have a clear structure for the metrics namespace.

## Non-goals

Add new metrics, or change the algorithms for existing ones.

## Current situation

Currently, the metrics namespace in `explainability-core` consist of a single package (`org.kie.trustyai.explainability.metrics`) with a single class (`FairnessMetrics`).

## Proposal

The proposed metrics namespace is as follows:


    org.kie.trustyai.explainability.metrics.fairness
    ├── group
    │   ├── statistical_parity
    │   ├── disparate_impact
    │   ├── equalized_odds
    │   ├── equal_opportunity
    │   └── conditional_demographic_disparity
    ├── individual
    │   ├── consistency
    │   └── ...
    ├── overall
    │   ├── gini_index
    │   └── ...
    ├── ranking
    │   ├── kendall_tau_distance
    │   └── ...
    ├── utility
    │   ├── fairness_through_awareness
    │   └── ...
    ├── composite
    │   └── ...
    └── external
        ├── provider1
        │   ├── algorithm1
        │   └── algorithm2
        ├── provider2
        │   ├── algorithm1
        │   └── algorithm2
        └── provider3
            ├── algorithm1
            └── algorithm2

Externally provided metrics can have their Java entry point in the `external` package, and the actual implementation in a sub-package. Separating by provider will allow for name clashes and conflicts to be resolved.


### Threat model

N/A

## Alternatives Considered / Rejected

N/A

## Challenges

This structure will have to duplicated in the Python bindings, as such the merge will have to be coordinated with an update to the Python bindings.

## Dependencies

See "Challenges".

## Consequences if not completed

The metrics namespace will remain as is, which will make it harder to navigate and understand.
