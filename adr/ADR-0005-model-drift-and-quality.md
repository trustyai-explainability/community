---
num: 5 # allocate an id when the draft is created
title: Model Drift and Quality Metrics
status: "Draft" # One of Draft, Accepted, Rejected
authors:
  - "robgeada" # One item for each author, as github id or "firstname lastname"
tags:
  - "core,java,service" # e.g. service, python, java, etc
---

## Title
Model Drift and Quality Metrics

## Definitions
**Data drift**: The evolution of the distribution of inference data passing through a model over time, 
such that the newer inbound data has "drifted" away from the original training data of the model. 
For example. imagine a model that processes sensor data in a production line, and was trained over
a collection of readings over a specific time window. Over time, the production line replaces and changes
the sensors, but does not retrain the model. The inbound readings from the new sensors will likely be
different from what the model was trained on; the data has _drifted_ away from the original 
training distribution. 

**Concept drift**: The change of the desired or "correct" inference *output* over time, as compared
to the original training data. For example, imagine a model trained to predict credit scores given
user data. If the credit bureau changes their algorithm to assign credit scores, the model's learned
mapping will no longer be relevant; the _concept_ has drifted away from the model's concept of the task.

**Model Quality**: the qualitative performance of the model on the task at hand. For classification,
this could be classification accuracy ("what percentage of the predicted classes are correct?"), for 
regression it might be an aggregated distance function (e.g., "what is the average mean square error
of the predictions?"), but can be arbitrarily defined given the model, data, and task at hand. 

**Model drift**: the degradation of model quality due to changes in the data or desired task over time.
Cause of model drift can be data or concept drift, for example. 


## Context and Problem Statement
Being able to monitor various kinds of drift is very useful during a model deployment to identify when
the model should be retrained to incorporate the new inbound data. However, measuring these various drifts 
can be quite specific to the model and data in question, and thus universal support for _any_ kind 
of drift detection is quite difficult. 

Furthermore, concept drift and model quality require a source of _ground truth_, that is, a labeling
of the "correct" value for each model inference. For many tasks, this requires some form of expert-in-the-loop
to manually ascertain these correct values for each inference.

## Goals
- Refactor the TrustyAI metrics service to better generalize to new metrics and metric types
  - This opens the door to integrate IBM data/model drift metrics
- Add an endpoint that allows for the assignment of "ground-truth" labels to existing inferences, for quality metrics
- Add the ability to tag certain datapoints within the collected TrustyAI data as "training" data for reference in
data drift metrics
- Add a very generalized data/concept drift metric for numeric datasets, as an MVP
- Use data-drift as a proxy metric for model quality

## Non-goals
- Will not be implementing _general_ support for any metric

## Current situation
TrustyAI currently has no provision for data drift, concept drift, or model quality metrics. The two fairness 
metrics are currently hardcoded into the service, and each new metric would have to be similarly hardcoded.

## Proposal
### Metric refactoring:
- Align metric serving with the proposed ADR-0002 
- Allow for easy addition of new metrics into this schema
- Generalize the metric endpoint + prometheus publishing to allow for a generic metric class, where
      each individual metric is an implementations/subclass of this metric
  - Likely, there will need to be some structure of "GenericMetric" -> "GenericMetricSubClass" -> "SpecificMetricClass",
  such as "GenericMetric" -> "GenericFairnessMetric" -> "SPD", where each is inheriting/extending the previous class. 
- This will allow for easy integration of IBM/community metrics
### Ground truth labeling endpoint:
Create endpoint that accepts a map of ground-truth labels
- This can have two possible schemas:
    ```json
    {
     "ModelID": modelID,
     "Output_0": {
        "PredictionID_0": y_0_0,
        "PredictionID_1": y_1_0,
        ... 
        "PredictionID_n": y_n_0,
      }, 
      "Output_1": {...},
      ...
      "Output_N": {...},
    }
    ```
    or 
  ```json
    {
     "ModelID": modelID,
     "PredictionID_0": {
        "Output_0": y_0_0,
        "Output_1": y_0_1,
        ... 
        "Output_n": y_0_n,
      }, 
      "PredictionID_1": {...},
      ...
      "PredictionID_N": {...},
    }
    ```
  We can either pick one of these, or support both, shouldn't be too hard to allow both.
        
### Training data labeling endpoint
Endpoint that accepts a request of schema:
  ```json
    {
     "ModelID": modelID,
     "label":  $label-to-assign,
     "datapoints": [PredictionID_0, PredictionID_1, ..., PredictionID_n]
    }
   ```
A dataset metadata label=$label-to-assign is assigned to each of the listed predictions, to allow for 
extraction of the training data within data analysis metrics. This is similar in function to the explainer endpoints, 
where synthetic datapoints receive specific labels to exclude them for fairness metrics. 

### Data Drift via Distribution Analysis Metric:
For purely numeric datasets, a rudimentary drift metric would be to calculate/receive the mean and skew
of each feature/output values over the training set. For each inbound datapoint, the distance of each feature/output
from the means can be reported in terms of standard deviation. For example, given some prediction
   
`f([x_0, x_1, ..., x_n]) = [y_0, y_1]`, we can report `[s_x_0, s_x_1, ..., s_x_n,  s_y_0, s_y_1]` as a measure
of how far each of those features and outputs are from their training distributions. These could be aggregated
in some way (mean, median, max) and reported as a single metric, or each reported as individual metrics. Then, users
could define thresholds to what consistutes "acceptable drift", for example, 1.5 standard deviations. 

The easiest implementation of this would be to ask for users to provide the expected means and skews, in the fields of a metric
request endpoint:
```json
POST /metrics/data/simpleDistributionAnalysis -d 
{
  "ModelID": modelID,
  "Feature": {
    "Means": {
      "Feature_0": u_0,
      "Feature_1": u_1,
      ...
      "Feature_n": u_1
    },
    "Skews": {
      "Feature_0": k_0,
      "Feature_1": k_1,
      ...
      "Feature_n": k_1
    }
  },
  "Output": {
      "Output_0": u_0,
      "Output_1": u_1,
      ...
      "Output_n": u_1
    },
    "Skews": {
      "Output_0": k_0,
      "Output_1": k_1,
      ...
      "Output_n": k_1
    }
}
```
however, these could also be calculated automatically by analyzing any data labeled as "training" within the captured inference data:
```json
POST /metrics/data/simpleDistributionAnalysis -d 
{
  "ModelID": modelID,
}
```

### Data Drift as a Proxy for Model Quality
Models are unlikely to perform well on data that has drifted far from the original training distributions. As such, a data drift
metric, aggregating total drift over the last _n_ predictions, is likely a good proxy for the model _quality_ over that same window, 
and therefore we can provide an indicator of model quality _without requiring the laborious hand-labeling of inference ground-truths_

### Threat model

N/A

## Alternatives Considered / Rejected

N/A

## Challenges
- Metric refactoring may be a complex task to create a generic metric model
- Retrieving data points with specific labels, as well as labelling specific datapoints by ID, is difficult
given the lack of proper database support within TrustyAI

## Dependencies

See "Challenges".

## Consequences if not completed
Can not create model quality metrics, or monitor data.
