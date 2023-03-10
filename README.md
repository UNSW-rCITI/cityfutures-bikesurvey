UNSW City Futures - Smartphone cyclist travel survey 2023
================

- <a href="#requirements" id="toc-requirements">Requirements</a>
- <a href="#data-extraction" id="toc-data-extraction">Data extraction</a>
- <a href="#data-dictionary" id="toc-data-dictionary">Data dictionary</a>
  - <a href="#stage_uuids" id="toc-stage_uuids"><code>Stage_uuids</code></a>
  - <a href="#stage_profiles"
    id="toc-stage_profiles"><code>Stage_profiles</code></a>
  - <a href="#stage_anaylsis_timeseries"
    id="toc-stage_anaylsis_timeseries"><code>Stage_anaylsis_timeseries</code></a>
    - <a href="#analysisrecreated_location"
      id="toc-analysisrecreated_location"><code>analysis/recreated_location</code></a>
    - <a href="#analysisconfirmed_trip"
      id="toc-analysisconfirmed_trip"><code>analysis/confirmed_trip</code></a>

This document serves as an initial analysis and data dictionary for
Fourstep.

# Requirements

To run this document you need:

- Quarto 1.2 or newer,
- R 4.0 or newer, and
- [Docker](https://www.docker.com/products/docker-desktop/) or MongoDB
  4.4.0 (tested).

# Data extraction

Fourstep uses [MongoDB](https://www.mongodb.com/), a
[NoSQL](https://en.wikipedia.org/wiki/NoSQL) database, to store its data
and user data uploaded from their mobile phone devices. First you need
to restore the database from the backup file to a running instance of
MongoDB. If you have Docker installed, you can run the following command
to start a MongoDB instance:

``` bash
docker run -d -p 27017:27017 --name test-mongodb mongo:4.4.0
```

Then you can restore the database from the backup file using the
following command:

``` bash
docker cp '<path/to/fourstep-mongdb-backup.tar.gz>' test-mongodb:/dump.tar.gz
docker exec test-mongodb sh -c 'mongorestore --drop --gzip --archive=dump.tar.gz'
```

Now we use R to connect to the database and extract the data we need.

``` r
# install these required packages if you don't have them with :
# `install.packages(c("mongolite", "jsonlite", "dplyr", "data.table", "magrittr", "lubridate"))`
library(data.table)
library(magrittr)
library(mongolite)
library(jsonlite)
library(dplyr)
library(lubridate)
```

The code chunk below shows how to connect to the running instance of our
MongoDB database.

``` r
# Names of the collections in the database
COLLECTIONS <-
  c(
    "Stage_Profiles",
    "Stage_analysis_timeseries",
    "Stage_pipeline_state",
    "Stage_timeseries",
    "Stage_timeseries_error",
    "Stage_updateable_models",
    "Stage_usercache",
    "Stage_uuids"
  )

# helper function to create a connection to the database
create_connection <- function(db, collection, url) {
  stopifnot(collection %in% COLLECTIONS)
  mongolite::mongo(db = db,
                   collection = collection,
                   url = url)
}

# helper function to create connections to the database
connect_stage_collections <-
  function(db = "Stage_database",
           collections = COLLECTIONS,
           url = "mongodb://localhost:27017") {
    cons <-
      lapply(collections, function(x)
        create_connection(
          db = db,
          collection = x,
          url = url
        ))
    names(cons) <- collections
    cons
  }

# create connections to the database
conn <- connect_stage_collections()

# helper function to convert UUID to string
mutate_uuid_to_string <- function(.data, uuid_col) {
  .data %>%
    dplyr::rowwise() %>%
    dplyr::mutate({{uuid_col}} := paste0(unlist({{uuid_col}}), collapse = "")) %>%
    dplyr::ungroup()
}
```

The following R code can be used to extract the data from the database
and save them to CSV files.

``` r
uuids <- conn$Stage_uuids$find() %>%
  mutate_uuid_to_string(uuid) %>%
  jsonlite::flatten()

profiles <- conn$Stage_Profiles$find() %>%
  mutate_uuid_to_string(user_id) %>%
  jsonlite::flatten() 

confirmed_trips <- conn$Stage_analysis_timeseries$find(query = '
  {
    "metadata.key": "analysis/confirmed_trip"
  }
  ') %>% 
  mutate_uuid_to_string(user_id) %>%
  jsonlite::flatten()

recreated_locations <- conn$Stage_analysis_timeseries$find(query = '
  {
    "metadata.key": "analysis/recreated_location"
  }
  ') %>%
  mutate_uuid_to_string(user_id) %>%
  jsonlite::flatten()

readr::write_csv(uuids, here::here("data", "uuids.csv"))
readr::write_csv(profiles, here::here("data", "profiles.csv"))
readr::write_csv(confirmed_trips, here::here("data", "confirmed_trips.csv"))
readr::write_csv(recreated_locations, here::here("data", "recreated_locations.csv"))
```

The code chuck below shows how to match the confirmed trips with the
recreated locations which are the GPS traces collected from users???
devices. Matching them together allows us to get the XYZ trajectory for
each confirmed trip along with the measured speed and sensed motion
activity at each GPS point.

``` r
# set locations as data.table and duplicate location time to set an "interval"
data.table::setDT(recreated_locations) %>%
  .[, data.fmt_time := lubridate::as_datetime(data.fmt_time)] %>%
  .[, data.fmt_time_dummy := (data.fmt_time)]
data.table::setkey(recreated_locations, user_id, data.fmt_time, data.fmt_time_dummy)

# join locations to trips according to time interval overlap (all done in UTC time for consistency)
data.table::setDT(confirmed_trips)
confirmed_trips[, `:=`(
  data.start_fmt_time = lubridate::as_datetime(data.start_fmt_time),
  data.end_fmt_time = lubridate::as_datetime(data.end_fmt_time)
)]
data.table::setkey(confirmed_trips, user_id, data.start_fmt_time, data.end_fmt_time)

# Here is an example of how you can merge the confirmed_trips table and the recreated locations table
# together to get a table of locations (trip trajectories) that are associated with confirmed trips. 
joined_trips_and_locs <- data.table::foverlaps(
  recreated_locations[, .(user_id, data.fmt_time, data.fmt_time_dummy, data.mode, data.idx)], 
  confirmed_trips[, .(user_id, data.start_fmt_time, data.end_fmt_time)],
  type = "any", 
  nomatch = NULL
) %>%
.[, data.fmt_time_dummy := NULL]
```

# Data dictionary

This section provides information about each MongoDB collection of
Fourstep that we extracted in the previous section.

## `Stage_uuids`

This collection contains information about users, including their unique
identifier (`user_id`), email address (`user_email`), and the timestamp
when the user was last updated (`update_ts`). This table can be used for
tracking user activity or updating user information in a database.

| Field name | Data type | Description                                                |
|------------|-----------|------------------------------------------------------------|
| user_email | string    | The email address of the user.                             |
| user_id    | string    | The unique identifier of the user.                         |
| update_ts  | string    | The timestamp when the user was last logged in to the app. |

## `Stage_profiles`

The table contains information about a user and their device, including
app usage and device information. Here is a data dictionary describing
the fields:

| Variable Name      | Data Type | Description                                                                                                               |
|--------------------|-----------|---------------------------------------------------------------------------------------------------------------------------|
| user_id            | String    | A unique identifier for the user.                                                                                         |
| mpg_array          | Float     | A value representing fuel efficiency of a vehicle in miles per gallon.                                                    |
| source             | String    | The source of the data, likely referring to the app or service that collected it.                                         |
| update_ts          | Datetime  | A timestamp indicating when the data was last updated.                                                                    |
| client_app_version | String    | The version of the app that the user is running.                                                                          |
| client_os_version  | String    | The version of the operating system on the user???s device.                                                                 |
| curr_platform      | String    | The platform (e.g.?????ios???, ???android???) of the user???s device.                                                                |
| manufacturer       | String    | The manufacturer of the user???s device.                                                                                    |
| phone_lang         | String    | The language setting on the user???s device.                                                                                |
| client             | String    | A value indicating the client type (e.g.?????mobile???, ???web???).                                                                |
| curr_sync_interval | Integer   | The interval (in seconds) at which the user???s data is synchronized.                                                       |
| device_token       | String    | A unique token assigned to the user???s device, which may be used for push notifications or other device-specific features. |

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

### `analysis/recreated_location`

This table can be used for analyzing location data of a user. It
contains information about the latitude and longitude of the location,
the timestamp of the location, the altitude, distance, speed, heading,
and accuracy. It also has information about the mode and section of the
location, the filter used for obtaining the data, and the sensed speed.
This data can be used for understanding user mobility patterns,
transportation choices, and activity recognition.

| Field Name                       | Data Type | Description                                                                                                                                                                |
|----------------------------------|-----------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| user_id                          | string    | Unique identifier for the user.                                                                                                                                            |
| metadata.key                     | string    | Metadata key.                                                                                                                                                              |
| metadata.platform                | string    | Platform for the metadata.                                                                                                                                                 |
| metadata.write_ts                | float     | Timestamp for when the metadata was written.                                                                                                                               |
| metadata.time_zone               | string    | Timezone of the metadata.                                                                                                                                                  |
| metadata.write_fmt_time          | string    | Time in which the metadata was written in string format.                                                                                                                   |
| metadata.write_local_dt.year     | integer   | The year in which the metadata was written.                                                                                                                                |
| metadata.write_local_dt.month    | integer   | The month in which the metadata was written.                                                                                                                               |
| metadata.write_local_dt.day      | integer   | The day in which the metadata was written.                                                                                                                                 |
| metadata.write_local_dt.hour     | integer   | The hour in which the metadata was written.                                                                                                                                |
| metadata.write_local_dt.minute   | integer   | The minute in which the metadata was written.                                                                                                                              |
| metadata.write_local_dt.second   | integer   | The second in which the metadata was written.                                                                                                                              |
| metadata.write_local_dt.weekday  | integer   | The weekday in which the metadata was written.                                                                                                                             |
| metadata.write_local_dt.timezone | string    | The timezone in which the metadata was written.                                                                                                                            |
| latitude                         | float     | The latitude of the location.                                                                                                                                              |
| longitude                        | float     | The longitude of the location.                                                                                                                                             |
| ts                               | float     | The timestamp of the location.                                                                                                                                             |
| fmt_time                         | string    | The formatted time of the location.                                                                                                                                        |
| altitude                         | float     | The altitude of the location.                                                                                                                                              |
| distance                         | float     | The distance of the location.                                                                                                                                              |
| speed                            | float     | The speed of the location.                                                                                                                                                 |
| heading                          | float     | The heading of the location.                                                                                                                                               |
| idx                              | integer   | The index of the location.                                                                                                                                                 |
| mode                             | string    | The mode of the location. See https://github.com/e-mission/e-mission-server/blob/6b48957818444d8514b510f7a36c1c4c8e9b29e2/emission/core/wrapper/motionactivity.py#L12-L46. |
| section                          | string    | The section of the location.                                                                                                                                               |
| accuracy                         | float     | The accuracy of the location.                                                                                                                                              |
| elapsedRealtimeNanos             | float     | Elapsed time since boot, including time spent in sleep.                                                                                                                    |
| filter                           | string    | The filter of the location.                                                                                                                                                |
| sensed_speed                     | float     | The sensed speed of the location.                                                                                                                                          |
| floor                            | float     | The floor of the location.                                                                                                                                                 |
| vaccuracy                        | float     | The vertical accuracy of the location.                                                                                                                                     |
| local_dt_year                    | integer   | The year of the location in local time.                                                                                                                                    |
| local_dt_month                   | integer   | The month of the location in local time.                                                                                                                                   |
| local_dt_day                     | integer   | The day of the location in local time.                                                                                                                                     |
| local_dt_hour                    | integer   | The hour of the location in local time.                                                                                                                                    |
| local_dt_minute                  | integer   | The minute of the location in local time.                                                                                                                                  |
| local_dt_second                  | integer   | The second of the location in local time.                                                                                                                                  |
| local_dt_weekday                 | integer   | The weekday of the location in local time.                                                                                                                                 |
| local_dt_timezone                | string    | The timezone of the location in local time.                                                                                                                                |
| loc.type                         | string    | The type of the location.                                                                                                                                                  |
| loc.coordinates                  | string    | The coordinates of the location in string                                                                                                                                  |

### `analysis/confirmed_trip`

This dataset contains information about confirmed trips. The data
includes metadata about the trip such as the platform, the timestamp
when the data was written, the time zone, and the source of the data. It
also contains information about the duration, distance, start and end
locations, and the cleaned and inferred trip labels. Additionally, the
table has information about user input and confidence thresholds. This
data can be used for analysing user transportation choices and activity
recognition.

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
