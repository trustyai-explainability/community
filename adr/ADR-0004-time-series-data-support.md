---
num: 4
title: Time-series data support
status: "Draft"
authors:
  - "ruivieira"
tags:
  - "core,python,service"
---

## Title

Time-Series Data Support

## Context and Problem Statement

TrustyAI's explainability core presently incorporates `LocalExplainer`[^local] and `GlobalExplainer`[^global] interfaces to provide explanations for predictions.
Additionally, the TrustyAI service provides a REST API that allows users to request local and gobal explanations for provided predictions.

[^local]: [explainability-core/src/main/java/org/kie/trustyai/explainability/local/LocalExplainer.java](https://github.com/trustyai-explainability/trustyai-explainability/blob/72cfb348c9ac272ac3fa41637431648aff41b483/explainability-core/src/main/java/org/kie/trustyai/explainability/local/LocalExplainer.java)
[^global]: [explainability-core/src/main/java/org/kie/trustyai/explainability/global/GlobalExplainer.java](https://github.com/trustyai-explainability/trustyai-explainability/blob/72cfb348c9ac272ac3fa41637431648aff41b483/explainability-core/src/main/java/org/kie/trustyai/explainability/global/GlobalExplainer.java)

While these interfaces and endpoints are effective for single data point predictions, they are not designed to handle explanations for time-series data where predictions are interconnected and can (potentially) consist of multiple data points.

## Goals

- Introduce a dedicated interface for time-series data, extending the existing ones.
- Introduce a dedicated endpoint for time-series algorithms, extending ones.
- Enable the application to effectively explain predictions in time-series data.

## Non-goals

- This proposal does not aim at changing the functionality of existing `LocalExplainer` or `GlobalExplainer` interfaces.
- The proposal does not aim at introducing any specific time-series explanation algorithms implementations.

## Current situation

Currently, the explainability-core[^1] handles and explains point-wise predictions using the `LocalExplainer` and `GlobalExplainer` interfaces. Support exists for these classes of explainers at the TrustyAI service level.

[^1]: https://github.com/trustyai-explainability/trustyai-explainability

However, neither the service nor explainability-core support the explanation of time-series data predictions where the data points have temporal dependencies.

## Proposal

We propose to introduce a new interface, `TimeSeriesExplainer`. This interface will be designed to handle a list of predictions in time-series data and provide an explanation that takes into account the temporal dependencies. It will extend from the `LocalExplainer` interface, inheriting its explanation signature.

```java
public interface TimeSeriesExplainer<T extends TimeSeriesExplanation> extends LocalExplainer<T> {

    CompletableFuture<T> explainAsync(List<Prediction> predictions, PredictionProvider model);

    CompletableFuture<T> explainAsync(List<Prediction> predictions, PredictionProvider model, Consumer<T> intermediateResultsConsumer);

}
```

The `TimeSeriesExplanation` is a new type of explanation (introduced here as a marker interface) specifically designed for time-series data. Future time-series explanations types can extend from this interface.

As an example of the explainability-core usage of these interfaces, we can consider the following scenario:
Suppose we have the following data representing our time-series data as

```text
timestamp, f1, f2, f3, y
2023-01-01 00:00:00, 1.1, 2.2, 3.3, 4
2023-01-02 00:00:00, 2.2, 3.3, 4.4, 5
2023-01-03 00:00:00, 3.3, 4.4, 5.5, 6
2023-01-04 00:00:00, 4.4, 5.5, 6.6, 7
2023-01-05 00:00:00, 5.5, 6.6, 7.7, 8
2023-01-06 00:00:00, 6.6, 7.7, 8.8, 9
2023-01-07 00:00:00, 7.7, 8.8, 9.9, 10
2023-01-08 00:00:00, 8.8, 9.9, 11.0, 11
2023-01-09 00:00:00, 9.9, 11.0, 12.1, 12
2023-01-10 00:00:00, 11.0, 12.1, 13.2, 13
```

We will convert this data into a `List<Prediction>`. Each prediction corresponds to a row in the table, with `timestamp`, `f1`, `f2`, and `f3` as features and `y` as the output:

```java
List<OffsetDateTime> timestamps = Arrays.asList(
    OffsetDateTime.parse("2023-01-01T00:00:00Z"),
    OffsetDateTime.parse("2023-01-02T00:00:00Z"),
    OffsetDateTime.parse("2023-01-03T00:00:00Z"),
    OffsetDateTime.parse("2023-01-04T00:00:00Z"),
    OffsetDateTime.parse("2023-01-05T00:00:00Z"),
    OffsetDateTime.parse("2023-01-06T00:00:00Z"),
    OffsetDateTime.parse("2023-01-07T00:00:00Z"),
    OffsetDateTime.parse("2023-01-08T00:00:00Z"),
    OffsetDateTime.parse("2023-01-09T00:00:00Z"),
    OffsetDateTime.parse("2023-01-10T00:00:00Z")
);

List<Double> f1s = Arrays.asList(1.1, 2.2, 3.3, 4.4, 5.5, 6.6, 7.7, 8.8, 9.9, 11.0);
List<Double> f2s = Arrays.asList(2.2, 3.3, 4.4, 5.5, 6.6, 7.7, 8.8, 9.9, 11.0, 12.1);
List<Double> f3s = Arrays.asList(3.3, 4.4, 5.5, 6.6, 7.7, 8.8, 9.9, 11.0, 12.1, 13.2);
List<Integer> ys = Arrays.asList(4, 5, 6, 7, 8, 9, 10, 11, 12, 13);

List<Prediction> predictions = new ArrayList<>();

for (int i = 0; i < ys.size(); i++) {
    PredictionInput input = new PredictionInput(Arrays.asList(
            FeatureFactory.newTimeFeature("timestamp", timestamps.get(i)),
            FeatureFactory.newNumericalFeature("f1", f1s.get(i)),
            FeatureFactory.newNumericalFeature("f2", f2s.get(i)),
            FeatureFactory.newNumericalFeature("f3", f3s.get(i))
    ));

    PredictionOutput output = new PredictionOutput(Arrays.asList(
            new Output("y", Type.NUMBER, new Value<>(ys.get(i)), 1.0)
    ));

    predictions.add(new SimplePrediction(input, output));
}

TimeSeriesExplainer<TimeSeriesExplanation> explainer = /* Instance of some implementation of TimeSeriesExplainer */;
CompletableFuture<TimeSeriesExplanation> explanation = explainer.explainAsync(predictions, someModel);
```

This example shows how we convert a time-series data set into a (basic) `List<Prediction>` and use an instance of `TimeSeriesExplainer` to generate an explanation for the time-series prediction. The specifics of the `TimeSeriesExplainer` implementation and the model are outside the scope of this proposal.

In addition to the earlier use case, it's important to highlight that this interface design, due to its dual dimensions - temporal (as elements in the list) and features (as list of features) - allows to accomodate for both univariate and multivariate time-series. This makes the interface flexible and robust for different kinds of time-series data.

### TimeSeriesDataframe Extension

We also propose to extend the `org.kie.trustyai.explainability.model.Dataframe` data structure to provide a `TimeSeriesDataframe`. The `TimeSeriesDataframe` will be specifically designed to handle time-series data, providing functionalities that are relevant for time-series data and analysis.

The `TimeSeriesDataframe` could be created from the provided data as follows:

```java
TimeSeriesDataframe tsdf = TimeSeriesDataframe.createFrom(predictions, "timestamp");
```

In the above example, "timestamp" is a column in the data that represents the temporal index. The `createFrom` method will create a `TimeSeriesDataframe` using this column as the time index.

The `TimeSeriesDataframe` will also include methods specific to time-series data handling. This includes (but not limited to) operations as:

- **Resampling**: Changing the frequency of the data points (*e.g.*, from minutes to hours).
- **Rolling Window operations**: Applying statistical functions over a sliding window of data points (*e.g.*, a rolling mean or standard deviation).
- **Time Shifting**: Shifting the data points forward or backward in time.
- **Handling Missing Data**: Algorithms to fill in missing data points in the time series.

```java
// Resample the data to a lower frequency
tsdf.resample("1H");

// Calculate a rolling mean with a window size of 3
tsdf.rollingMean(3);

// Shift the data points 2 steps forward
tsdf.shift(2);

// Fill missing data points using linear interpolation
tsdf.interpolate(INTERPOLATION.LINEAR);
```

### Service Endpoints Extension

To integrate time-series explanations into our existing service, we propose a new endpoint following the structure of our current *explainer* endpoints:

`/explainers/timeseries/<algorithm>`

This would align with the existing service structure:

- `/explainers/local/<algorithm>`
- `/explainers/global/<algorithm>`
- `/metrics/<algorithm>`

The HTTP POST payload will be:

```java
public class TimeSeriesRequest {
    private String modelId;
    private Optional<List<Date>> timestamp; // optional field
    private Map<String, List<Double>> inputs;
    private Map<String, List<Double>> outputs;
}
```

The timestamp field is optional. When not present, we assume the data points are not timestamped and that the data points in the inputs and outputs maps are sequential in the time-series (with no specific time information).

Diagram showing how the inputs, outputs and timestamp map to the `Prediction` objects and the `TimeSeriesExplainer` interface:

```text
TimeSeriesRequest
   |
   |__ timestamp (Optional List of dates representing the temporal index)
   |
   |__ inputs (Map: Feature -> List of feature values over time)
   |
   |__ outputs (Map: Output -> List of output values over time)
   |
   V
List<Prediction> (Prediction comprises of a PredictionInput and a PredictionOutput)
   |
   V
TimeSeriesExplainer<T> (T can be a type specific to time-series explanation)
```

Where each key in the inputs and outputs map corresponds to a feature or output name.

#### Case with Timestamp

Given the example data:

| timestamp | f1  | f2  | f3  | y   |
| --------- | --- | --- | --- | --- |
| 1/1/2023  | 0.1 | 0.2 | 0.3 | 0.5 |
| 2/1/2023  | 0.2 | 0.3 | 0.4 | 0.6 |
| 3/1/2023  | 0.3 | 0.4 | 0.5 | 0.7 |
| 4/1/2023  | 0.4 | 0.5 | 0.6 | 0.8 |
| 5/1/2023  | 0.5 | 0.6 | 0.7 | 0.9 |
| 6/1/2023  | 0.6 | 0.7 | 0.8 | 1.0 |
| 7/1/2023  | 0.7 | 0.8 | 0.9 | 1.1 |
| 8/1/2023  | 0.8 | 0.9 | 1.0 | 1.2 |
| 9/1/2023  | 0.9 | 1.0 | 1.1 | 1.3 |
| 10/1/2023 | 1.0 | 1.1 | 1.2 | 1.4 |

The corresponding JSON payload for the `TimeSeriesRequest` would be:

```json
{
  "modelId": "model123",
  "predictionId": "prediction123",
  "timestamp": [
    "2023-01-01T00:00:00Z",
    "2023-02-01T00:00:00Z",
    "2023-03-01T00:00:00Z",
    "2023-04-01T00:00:00Z",
    "2023-05-01T00:00:00Z",
    "2023-06-01T00:00:00Z",
    "2023-07-01T00:00:00Z",
    "2023-08-01T00:00:00Z",
    "2023-09-01T00:00:00Z",
    "2023-10-01T00:00:00Z"
  ],
  "inputs": {
    "f1": [0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1.0],
    "f2": [0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1.0, 1.1],
    "f3": [0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1.0, 1.1, 1.2]
  },
  "outputs": {
    "y": [
      0.5, 0.6, 0.7, 0.8, 0.9, 1.0, 1.1, 1.2, 1.3,

      1.4
    ]
  }
}
```

#### Case without Timestamp

If the timestamp information is not provided, the same data would be:

| f1  | f2  | f3  | y   |
| --- | --- | --- | --- |
| 0.1 | 0.2 | 0.3 | 0.5 |
| 0.2 | 0.3 | 0.4 | 0.6 |
| 0.3 | 0.4 | 0.5 | 0.7 |
| 0.4 | 0.5 | 0.6 | 0.8 |
| 0.5 | 0.6 | 0.7 | 0.9 |
| 0.6 | 0.7 | 0.8 | 1.0 |
| 0.7 | 0.8 | 0.9 | 1.1 |
| 0.8 | 0.9 | 1.0 | 1.2 |
| 0.9 | 1.0 | 1.1 | 1.3 |
| 1.0 | 1.1 | 1.2 | 1.4 |

And the corresponding JSON payload for the `TimeSeriesRequest` would be:

```json
{
  "modelId": "model123",
  "predictionId": "prediction123",
  "inputs": {
    "f1": [0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1.0],
    "f2": [0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1.0, 1.1],
    "f3": [0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1.0, 1.1, 1.2]
  },
  "outputs": {
    "y": [0.5, 0.6, 0.7, 0.8, 0.9, 1.0, 1.1, 1.2, 1.3, 1.4]
  }
}
```

In both cases, the structure of the JSON object corresponds to the structure of the `TimeSeriesRequest` class as shown earlier. The presence or absence of the timestamp field distinguishes whether time information is provided.

### Storage

In the context of the `TimeSeriesExplainer`, the existing storage access is abstracted by the `StorageInterface` (for managing I/O operations) and `DataSource` (for controlling the data that `StorageInterface` returns). Both these abstractions are orchestrated by the `DataParser` which is responsible for converting the raw data into a `TimeSeriesDataframe` object.

In order to integrate time-series data into our existing infrastructure, we need to make the following changes:

#### `StorageInterface`

```java
public interface StorageInterface {
    
    // ... 

    ByteBuffer readTimeSeriesData(String modelId, String timestamp) throws StorageReadException;

    ByteBuffer readPredictionData(String modelId, String predictionId) throws StorageReadException;
}
```

#### `DataParser`

```java
public interface DataParser {
    
    // ...

    TimeSeriesDataframe toTimeSeriesDataframe(ByteBuffer inputs, Metadata metadata) throws DataframeCreateException;

    TimeSeriesDataframe toPredictionDataframe(ByteBuffer inputs, Metadata metadata) throws DataframeCreateException;
}
```

#### JSON Examples

In these examples, we'll consider that payloads include `modelId`, `timestamp` or `predictionId` as necessary.

**Example 1:** A payload for reading time-series data (single data-point)

```json
{
  "modelId": "model123",
  "timestamp": "2023-06-24T12:00:00Z"
}
```

**Example 2:** A payload for reading prediction data (single data-point)

```json
{
  "modelId": "model123",
  "predictionId": "pred456"
}
```

In both examples, `model123` and `pred456` are identifiers for the model and prediction, respectively. The timestamp in the first example is in ISO 8601 format.

#### Scenario 1: Payload contains `timestamps`

If the payload contains a series of `timestamps`, it means that we need specific time-series data corresponding to those *timestamps*.

JSON Payload example:

```json
{
  "modelId": "model123",
  "timestamps": ["2023-06-24T12:00:00Z", "2023-06-24T13:00:00Z", "2023-06-24T14:00:00Z"]
}
```

**Diagram:**

The follwing diagram illustrates the process of fetching data from the storage based on the **timestamps** provided in the payload.

```
 Time-Series Data(Storage)         |      StorageInterface      |   Timestamps Requested
-----------------------------------------------------------------------------------------------------------------
| ID | Feature 1 | Feature 2 | Timestamp | |                   |  |   Timestamp   |  Corresponding Data  |
-----------------------------------------------------------------------------------------------------------------
| 1  |   120     |  4300     |     1     | |                   |  |               |                     |
| 2  |   150     |  4500     |     2     | |  <--------------- |  |       2       |  [2, 150, 4500]     |
| 3  |   170     |  4550     |     3     | |                   |  |               |                     |
| 4  |   180     |  4600     |     4     | |  <--------------- |  |       4       |  [4, 180, 4600]     |
| 5  |   210     |  4650     |     5     | |                   |  |               |                     |
-----------------------------------------------------------------------------------------------------------------
```

We can add the following methods to the `StorageInterface`, in order to allows us for sequential data retrieval:

```java
ByteBuffer readDataBefore(String modelId, Date timestamp) throws StorageReadException;

ByteBuffer readDataBefore(String modelId, Date timestamp, int batchSize) throws StorageReadException;

ByteBuffer readDataAfter(String modelId, Date timestamp) throws StorageReadException;

ByteBuffer readDataAfter(String modelId, Date timestamp, int batchSize) throws StorageReadException;

ByteBuffer readDataBetween(String modelId, Date timestampStart, Date timestampEnd) throws StorageReadException;
```

The following methods provide the functionality:

- `readDataBefore`, read the entire data before the given timestamp
- `readDataBefore`, read the data before the given timestamp, but no more than the given batch size
- `readDataAfter`, read the entire data after the given timestamp
- `readDataAfter`, read the data after the given timestamp, but no more than the given batch size
- `readDataBetween`, read the data between the given timestamps

#### Scenario 2: Payload contains `predictionIds`

If the payload contains a series of `predictionIds`, it means that we need specific prediction data corresponding to those prediction IDs.

JSON Payload example:

```json
{
  "modelId": "model123",
  "predictionIds": ["pred456", "pred789", "pred012"]
}
```

**Diagram:**

This diagram illustrates the process of fetching data from the storage based on the prediction IDs provided in the payload.

```text
 Predictions Data(Storage)         |      StorageInterface      |   Prediction IDs Requested
-----------------------------------------------------------------------------------------------------------------
| ID | Feature 1 | Feature 2 | Prediction ID | |                   |  |  Prediction ID  |  Corresponding Data  |
-----------------------------------------------------------------------------------------------------------------
| 1  |   120     |  4300     |     Pred1     | |                   |  |                 |                     |
| 2  |   150     |  4500     |     Pred2     | |  <-------------  |  |      Pred2      |  [2, 150, 4500]     |
| 3  |   170     |  4550     |     Pred3     | |                   |  |                 |                     |
| 4  |   180     |  4600     |     Pred4     | |  <-------------  |  |      Pred4      |  [4, 180, 4600]     |
| 5  |   210     |  4650     |     Pred5     | |                   |  |                 |                     |
-----------------------------------------------------------------------------------------------------------------
```

To implement this, you could add methods to the `StorageInterface` like:

```java
ByteBuffer readDataBefore(String modelId, String predictionId) throws StorageReadException;

ByteBuffer readDataBefore(String modelId, String predictionId, int batchSize) throws StorageReadException;

ByteBuffer readDataAfter(String modelId, String predictionId) throws StorageReadException;

ByteBuffer readDataAfter(String modelId, String predictionId, int batchSize) throws StorageReadException;

ByteBuffer readDataBetween(String modelId, String predictionIdStart, String predictionIdEnd) throws StorageReadException;
```

The rationale for these methods is the same as in the previous scenario.

```text
 Time-Series Data(Storage)         |      StorageInterface      |   Timestamps Requested
-----------------------------------------------------------------------------------------------------------------
| ID | Feature 1 | Feature 2 | Timestamp | |                   |  |   Timestamp   |  Corresponding Data  |
-----------------------------------------------------------------------------------------------------------------
| 1  |   120     |  4300     |     1     | |                   |  |               |                     |
| 2  |   150     |  4500     |     2     | |  <--------------- |  |   2 - 4       |  [2, 150, 4500]     |
| 3  |   170     |  4550     |     3     | |  <--------------- |  |               |  [3, 170, 4550]     |
| 4  |   180     |  4600     |     4     | |  <--------------- |  |               |  [4, 180, 4600]     |
| 5  |   210     |  4650     |     5     | |                   |  |               |                     |
-----------------------------------------------------------------------------------------------------------------
```

### Threat model

No new threats are identified for this Architecture Decision Record.

## Alternatives Considered / Rejected

An alternative would be modifying the existing explainer interfaces to handle time-series data. However, this could result in a more complex interface design and potentially break changes in the existing implementations.
This is would also likely be a violation of the Interface Segregation Principle.

## Challenges

- The introduction of the `TimeSeriesExplainer` interface might increase the complexity of our system.
- Allowing explainability and data support for such a broad field such as time-series data might be challenging and not cover all possible use cases.

## Dependencies

Athough the interface introduces no external dependencies whatsoever and is an internal addition, the service endpoints will create a new API contract by defining new payloads and new REST endpoints.

## Consequences if not completed

If not implemented, TrustyAI will lack the ability to effectively explain time-series predictions.
