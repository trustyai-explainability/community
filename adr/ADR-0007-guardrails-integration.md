---
num: 7 # allocate an id when the draft is created
title: TrustyAI integration with Guardrails
status: "Draft" # One of Draft, Accepted, Rejected
authors:
  - "evaline-ju" # One item for each author, as github id or "firstname lastname"
tags:
  - "guardrails,operator" # e.g. service, python, java, etc
---

## Title

TrustyAI integration with Guardrails

## Context and Problem Statement

“Guardrails” is a stack of various components that enable standalone detections on text, as well as invoking detections on text generation server input (i.e. user prompts) and text generation server output (i.e. generated text).

Components of the "Guardrails" stack include:
* Orchestrator - available [open-sourced here](https://github.com/foundation-model-stack/fms-guardrails-orchestrator) / [API here](https://foundation-model-stack.github.io/fms-guardrails-orchestrator/)
     * This REST service is the main entry point for a user trying to get detection information, such as on their LLM inputs and/or outputs. It coordinates the calls to text generation servers (LLM servers), chunker server(s), and detector server(s), described below.
* Detectors / detector servers
     * These REST services provide detection information such as (but not limited to) classifications on text and are expected to conform to the [detectors API](https://foundation-model-stack.github.io/fms-guardrails-orchestrator/?urls.primaryName=Detector+API). 
* Chunkers / chunker servers
     * Chunkers can be thought of as servers that “chunk” the target modality. For text, this can look like tokenizers or sentence splitters. Currently to work with the orchestrator, these are expected to be GRPC servers, to support bidirectional streaming. Detectors can specify corresponding chunkers in configuration. 
     * [Proto for reference](https://github.com/foundation-model-stack/fms-guardrails-orchestrator/blob/main/protos/caikit_runtime_Chunkers.proto)

We generally have assumed the text generation server(s) are separate from the “Guardrails” stack i.e. deployed and maintained separately. The grpc text generation provider can be specified through orchestrator configuration. Currently, the orchestrator is directly compatible and tested with:
1.  TGIS grpc interface, [proto](https://github.com/IBM/text-generation-inference/blob/main/proto/generation.proto)
2.  [caikit-nlp](https://github.com/caikit/caikit-nlp) text generation interface

The Guardrails stack could provide TrustyAI users text detections for their generative AI applications.

## Goals

* Allow TrustyAI users to be able to get detections in their LLM applications in real-time
* Allow TrustyAI users to bring in other detectors (e.g. potentially toxic text detectors) into their applications


## Non-goals

* Building and providing official images for the open-sourced [orchestrator](https://github.com/foundation-model-stack/fms-guardrails-orchestrator), along with any image maintenance issues such as security scanning


## Current situation

Current implementations of chunker servers and detector servers are proprietary but the open-sourced orchestrator can work with any servers implementing compatible APIs.


## Proposal

The integration will include:
* Standing up an open-sourced Guardrails stack
    * This will require open-sourced versions of at least one running detector and potentially chunker, depending on the detector of choice.
* A controller is able to deploy / maintain a Guardrails stack via the TrustyAI operator
    * This will eventually allow RHOAI consumers access to the Guardrails stack. However, we want to make sure the features are toggle-able so a consumer, who may want only metrics (through `TrustyAIService` today) or only detections, is not bound to include other features.


## Alternatives Considered / Rejected

N/A

## Challenges

Potentially a clear differentiation of capabilities available for a TrustyAI user - will it be easy to tell when to use a "metric" or a "detection"?

## Dependencies

Available Guardrails stack components (listed above)

## Consequences if not completed

TrustyAI will have to find alternate solutions for detections on text generation applications
