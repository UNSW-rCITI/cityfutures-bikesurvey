UNSW City Futures - Smartphone cyclist travel survey 2023
================

- <a href="#requirements" id="toc-requirements">Requirements</a>
- <a href="#data-dictionary" id="toc-data-dictionary">Data dictionary</a>
  - <a href="#stage_anaylsis_timeseries"
    id="toc-stage_anaylsis_timeseries"><code>Stage_anaylsis_timeseries</code></a>
    - <a href="#analysisconfirmed_trip"
      id="toc-analysisconfirmed_trip"><code>analysis/confirmed_trip</code></a>

``` r
library(data.table)
```

This document serves as the data dictionary for Fourstep.

# Requirements

To run this document you need:

- Quarto 1.2 or newer.
- MongoDB 4.4.0 and restore your Fourstep database backup.

# Data dictionary

The sub-sections in this section provides information about each MongoDB
collection of Fourstep.

## `Stage_anaylsis_timeseries`

``` json
> db.getCollection("Stage_analysis_timeseries").distinct("metadata.key")
[
    "analysis/cleaned_place",
    "analysis/cleaned_section",
    "analysis/cleaned_stop",
    "analysis/cleaned_trip",
    "analysis/cleaned_untracked",
    "analysis/confirmed_trip",
    "analysis/expected_trip",
    "analysis/inferred_labels",
    "analysis/inferred_section",
    "analysis/inferred_trip",
    "analysis/recreated_location",
    "analysis/smoothing",
    "inference/labels",
    "inference/prediction",
    "segmentation/raw_place",
    "segmentation/raw_section",
    "segmentation/raw_stop",
    "segmentation/raw_trip",
    "segmentation/raw_untracked"
]
```

### `analysis/confirmed_trip`

Below is the documentation of each field in the object:

| Field Name                                              | Data Type | Description                                                                                       |
|---------------------------------------------------------|-----------|---------------------------------------------------------------------------------------------------|
| \_id                                                    | ObjectId  | Unique identifier for the document                                                                |
| user_id                                                 | LUUID     | Unique identifier for the user                                                                    |
| metadata.key                                            | string    | Key for metadata field                                                                            |
| metadata.platform                                       | string    | Platform where the data was collected                                                             |
| metadata.write_ts                                       | float     | Timestamp of when the data was written to the database                                            |
| metadata.time_zone                                      | string    | Timezone of the data                                                                              |
| metadata.write_fmt_time                                 | string    | Timestamp of when the data was written to the database in formatted string format                 |
| data.source                                             | string    | Source of the trip data                                                                           |
| data.end_ts                                             | float     | Timestamp of the end of the trip                                                                  |
| data.end_loc                                            | object    | Geolocation of the end of the trip                                                                |
| data.raw_trip                                           | ObjectId  | Unique identifier for the raw trip                                                                |
| data.start_ts                                           | float     | Timestamp of the start of the trip                                                                |
| data.start_loc                                          | object    | Geolocation of the start of the trip                                                              |
| data.duration                                           | float     | Duration of the trip in seconds                                                                   |
| data.distance                                           | float     | Distance of the trip in meters                                                                    |
| data.start_place                                        | ObjectId  | Unique identifier for the start place of the trip                                                 |
| data.end_place                                          | ObjectId  | Unique identifier for the end place of the trip                                                   |
| data.cleaned_trip                                       | ObjectId  | Unique identifier for the cleaned trip                                                            |
| data.inferred_labels                                    | list      | List of inferred labels for the trip                                                              |
| data.inferred_trip                                      | ObjectId  | Unique identifier for the inferred trip                                                           |
| data.expectation.to_label                               | boolean   | Whether the trip is expected to have a label                                                      |
| data.confidence_threshold                               | float     | Threshold for confidence of trip inference                                                        |
| data.expected_trip                                      | ObjectId  | Unique identifier for the expected trip                                                           |
| data.user_input.trip_user_input                         | object    | User input data for the trip                                                                      |
| data.user_input.trip_user_input.metadata.key            | string    | Key for metadata field for the trip user input                                                    |
| data.user_input.trip_user_input.metadata.platform       | string    | Platform where the trip user input data was collected                                             |
| data.user_input.trip_user_input.metadata.write_ts       | integer   | Timestamp of when the trip user input data was written to the database                            |
| data.user_input.trip_user_input.metadata.time_zone      | string    | Timezone of the trip user input data                                                              |
| data.user_input.trip_user_input.metadata.write_fmt_time | string    | Timestamp of when the trip user input data was written to the database in formatted string format |
| data.user_input.trip_user_input.data.label              | string    | Label for the trip                                                                                |
| data.user_input.trip_user_input.data.name               | string    | Name of the trip                                                                                  |
| data.user_input.trip_user_input.data.version            | integer   | Version of the trip user input data                                                               |
| data.user_input.trip_user_input.data.xmlResponse        | string    | XML response for the trip user input data                                                         |
| data.user_input.trip_user_input.data.jsonDocResponse    | object    | JSON response for the trip user input data                                                        |
| data.user_input.trip_user_input.data.start_ts           | float     | Timestamp of the start of the trip                                                                |
| data.user_input.trip_user_input.data.end_ts             | float     | Timestamp of the end of the trip                                                                  |
| data.user_input.trip_user_input.data.start_fmt_time     | string    | Timestamp of the start of the trip in formatted string format                                     |
| data.user_input.trip_user_input.data.end_fmt_time       | string    | Timestamp of the end of the trip in formatted string format                                       |
