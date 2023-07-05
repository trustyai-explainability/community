# Architecture Decision Records (ADRs)

- A template to create new ADRs is available [here](../templates/adr-template.md).
- Submit a pull request to add a new ADR to this repository.
  - Use the `adr` folder as the base path for the ADR.
  - Label the ADR
    - Use the `ADR` and `ADR/under-discussion` label for the pull request.
    - If the ADR is for a purely internal change[^1], use the `ADR/internal` label.
  - Name your ADR file using the following convention: `ADR-NNNN-title.md`, where `NNNN` is the next sequential number and `title` is a short, lower-case, dash-separated description of the ADR.
  - Place any external documents referenced by the ADR in the `adr/assets` folder.

[^1]: An internal change is one that does not affect the external API of the TrustyAI service or the deployment architecture.

# Current ADRs

[List of ADRs currently in discussion](https://github.com/trustyai-explainability/community/pulls?q=is%3Aopen+is%3Apr+label%3AADR%2Funder-discussion).

# Approved ADRs

* [ADR-0001: TrustyAI external library integration](ADR-0001-trustyai-external-library-integration.md)
* [ADR-0002: Metrics and XAI namespaces](ADR-0002-metrics-and-xai-namespaces.md)
* [ADR-0004: Time-series data support](ADR-0004-time-series-data-support.md)