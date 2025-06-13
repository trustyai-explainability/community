---
num: 8 # allocate an id when the draft is created
title: Local/builtin chunker and detector for Guardrails
status: "Draft" # One of Draft, Accepted, Rejected
authors:
  - "resoluteCoder" # One item for each author, as github id or "firstname lastname"
tags:
  - "guardrails,rust" # e.g. service, python, java, etc
---

## Title

Local/builtin chunker and detector for Guardrails

## Context and Problem Statement

Right now, guardrails interact with chunkers and detectors through an API. For straightforward chunking and detection tasks, it would be beneficial to not have to rely on a model. Additionally, this could decrease the time it would take to set up a development environment

## Goals

* Looking to increase performance for simple chunking/detectors
* Ability to decrease the time it would take to set up a development environment
* Ability to customize detectors and chunkers

## Non-goals

N/A

## Current situation

Since the TrustyAI algorithmic core is written in Java, as well as the TrustyAI service which provides all the ODH integration (ModelMesh payload consumption, data storage handling, Prometheus publishing and REST endpoints), the only way to add new explainers or metrics is to implement them in Java or add them as a dependency, if available as a Java library.
Although the Python bindings for TrustyAI exist, their purpose is to invoke the Java algorithms from Python and not the other way around.
At the moment there is no out of the box solution for TrustyAI to leverage the existing explainability libraries, which arguably are written for Python.

## Proposal

To preserve the integrity of the guardrails while introducing alternative options for chunkers and detectors, one proposal is to  create  separate API servers that can reside within the same pod as the guardrails. This setup would allow the guardrails to be configured to communicate with these local chunkers and detectors via localhost.

### Language

This proposal's scope requires languages capable of parsing string data and performing regex operations, with the ability to communicate via gRPC and HTTP.

Based on several benchmark tests, Rust appears to be an excellent choice. Not only is the guardrails project implemented in Rust, but its speed also aligns well with our needs.

Sources

* [text-parsing](https://blog.scanner.dev/serverless-speed-rust-vs-go-java-python-in-aws-lambda-functions/)
* [network](https://medium.com/star-gazers/benchmarking-low-level-i-o-c-c-rust-golang-java-python-9a0d505f85f7)

### API Libs

For API libraries,  Axum would be used for HTTP (built on Tokio) and Tonic for gRPC. Fortunately, the guardrails project already utilizes both of these internally.

* [axum](https://github.com/tokio-rs/axum)
* [tonic](https://github.com/hyperium/tonic)

## Alternatives Considered / Rejected

N/A

## Challenges

N/A

## Dependencies

N/A
