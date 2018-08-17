# Big Data Pipeline - Google

## Pre-Requirements
- (Personal) Google Account, gmail account is ok.
- Registered to Google Cloud Platform (GCP) and access to [Google Cloud Console](https://console.cloud.google.com/) 
- GCP Project created. 
- Following APIs needs to be enabled in [APIs & Services](https://console.cloud.google.com/apis)
  - BigQuery API
  - Google Cloud Pub/Sub API
  - Google Cloud Functions API
  - Google Cloud Storage

## BigQuery

Go to the [BigQuery Console](https://console.cloud.google.com/bigquery)

### Step 1 - Create new dataset

| Parameter     | Value      |
| ------------- | ---------- |
| Dataset ID    | `Workshop` |
| Data location | `US`       |

### Step 2 - Create new table

| Parameter    | Value              |
| ------------ | ------------------ |
| Source Data  | Create empty table |
| Table name   | `Traffic`          |

| Schema Field | Type        | Mode       |
| ------------ | ----------- | ---------- |
| Location     | `STRING`    | `NULLABLE` |
| Time         | `TIMESTAMP` | `NULLABLE` |
| Direction    | `INTEGER`   | `NULLABLE` |
| VehicleType  | `STRING`    | `NULLABLE` |
| Speed        | `INTEGER`   | `NULLABLE` |

### Step 3 - Import rows from csv file (Create new table)

| Parameter                    | Value                                                 |
| -----------------------------| ----------------------------------------------------- |
| Source Data                  | Create from source                                    |
| Location                     | File upload [traffic_data.csv](data/traffic_data.csv?raw=true) |
| Table name                   | `Traffic`                                             |
| Schema: Automatically detect | Checked                                               |
| Options: Write preference    | Append to table                                       |


### Example queries

#### Pori statistics by vehicle type

```SQL
SELECT VehicleType, MIN(Speed) AS Min, AVG(Speed) AS Avg, MAX(Speed) AS Max
FROM `Workshop.Traffic`
WHERE Location='Pori'
GROUP BY VehicleType
```

#### Statistics by location

```sql
SELECT Location, MIN(Speed) AS Min, AVG(Speed) AS Avg, MAX(Speed) AS Max, STDDEV(Speed)AS StdDev
FROM `Workshop.Traffic`
GROUP BY Location
```


#### Hourly comparison Pori vs Vihti

```sql
WITH Pori AS (
  SELECT EXTRACT(HOUR FROM Time) AS Hour, MIN(Speed) AS PoriMinSpeed, AVG(Speed) AS PoriAvgSpeed, MAX(Speed) AS PoriMaxSpeed
  FROM `Workshop.Traffic`
  WHERE Location='Pori'
  GROUP BY Hour) 
  (
  WITH Vihti AS (
    SELECT EXTRACT(HOUR FROM Time) AS Hour, MIN(Speed) AS VihtiMinSpeed, AVG(Speed) AS VihtiAvgSpeed, MAX(Speed) AS VihtiMaxSpeed
    FROM `Workshop.Traffic`
    WHERE Location='Vihti'
    GROUP BY Hour)
  SELECT Pori.Hour, PoriMinSpeed, VihtiMinSpeed, PoriAvgSpeed, VihtiAvgSpeed, PoriMaxSpeed, VihtiMaxSpeed 
  FROM Pori, Vihti
  WHERE Pori.Hour=Vihti.Hour
  ORDER BY Hour )
```


### Links
[Google BigQuery Console](https://bigquery.cloud.google.com)

[Query Syntax](https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax)

[Data Types](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types)

[Functions](https://cloud.google.com/bigquery/docs/reference/standard-sql/functions-and-operators)

## Data Studio

Go to [Data Studio](https://datastudio.google.com)

### Step 1 - Create new blank report 
- Create a new blank report
- Navigate through Welcome, Terms and Preferences
- Rename `Untitled Report` to `Traffic Report`

### Step 2 - Create new data source
- Click `Create new data source` button and select BigQuery
- Authorise
- Select your GCP project
- Select `Workshop` dataset and `Traffic` table created earlier in this workshop
- Click `CONNECT`

- Edit connection

| Index | Field        | Type                    | Aggregation       | Description       |
| ----- | ------------ | ----------------------- |------------------ | ----------------- |
| 1     | `Speed`      | `Number`                | **Average**       |                   |
| 2     | `VehicleType`| `Text`                  | None              |                   | 
| 3     | `Time`       | `Date (YYYYMMDD)`       | None              |                   |
| 4     | `Direction`  | `Number`                | Sum               |                   |
| 5     | `Location`   | `Text`                  | None              |                   |
- Click `ADD TO REPORT`

### Step 3 - Add Data Table 
- Add Data Table 
- Drag and drop fields in Data tab 

|  Dimension    |
| ------------- |
| `Location`    |
| `VehicleType` |

|  Metric       |
| ------------- |
| `Speed`       |


- Select sorting by clicking sort and secondary sorting fields 

 
### Step 4 - Add Bar Chart to the canvas
- Add Bar Chart 
- Drag and drop fields in Data tab 

|  Dimension            | Value           |
| --------------------- | --------------- |
| Dimensions            | `Vehicle Type`  |


|  Metric       |
| ------------- |
| `Speed`       |

- Edit Metric properties, click `Calendar/Pen` icon

|  Property    | Value           |
| ------------ | --------------- |
| Name         | `Max Speed`     |
| Aggregation  | `Max`           |

- Sort by Speed


### Step 5 - Create filters
- Add Filter control
- Drag and drop fields in Data tab 

|  Dimension    |
| ------------- |
| `Location`    |

|  Metric       |
| ------------- |
| `Speed`       |

- Edit Metric properties, click `Calendar/Pen` icon

|  Property    | Value           |
| ------------ | --------------- |
| Name         | `Count`         |
| Aggregation  | `Count`         |



#### - Add Date Filter control


### Step 6 - Explore your report

- `View` your report
- Change values in `Location` and `Date` drop downs
- You can share your report 

[Extra: Data Studio exercise 2](datastudio2.md)

### Links
[Data Studio ](https://cloud.google.com/data-studio/)

[Welcome to Data Studio](https://support.google.com/datastudio/?hl=en#topic=6267740)

## Pub/Sub

Go to the [Pub/Sub Console](https://console.cloud.google.com/cloudpubsub)

### Step 1 - Create new topic

| Parameter | Value     |
| --------- | --------- |
| Name      | `Traffic` |

### Links

[What is Cloud Pub/Sub](https://cloud.google.com/pubsub/docs/overview)

## Cloud Functions

Go to the [Cloud Functions Console](https://console.cloud.google.com/functions/)

### Step 1 - Create new function

| Parameter    | Value                  |
| ------------ | ---------------------- |
| Name         | `Traffic-Function`     |
| Trigger      | `Cloud Pub/Sub topic`  |
| Topic        | `Traffic`               |
| package.json | See Step 2             |
| index.js     | See Step 3             |

### Step 2 - Edit **package.json**

```json
{
  "name": "Traffic function",
  "version": "1.0",
  "dependencies": {
     "@google-cloud/bigquery": "1.2"
  }
}
```

### Step 3 - Edit **index.js**

```javascript
var BigQuery = require('@google-cloud/bigquery');

exports.subscribe = (event, callback) => {
  const pubSubMessage = Buffer.from(event.data.data, 'base64').toString();
  console.log('PubSub message: ' + pubSubMessage);

  jsonMessage = JSON.parse(pubSubMessage);

  const bigquery = new BigQuery();
  const dataset = bigquery.dataset('Workshop');
  const table = dataset.table('Traffic');

  table.insert({
      Location: jsonMessage.Location,
      Time: jsonMessage.Time,
      Direction: jsonMessage.Direction,
      VehicleType: jsonMessage.VehicleType,
      Speed: jsonMessage.Speed
  });

  console.log('Done');

  callback();
};
```

### Step 4 - Test sending Pub/Sub message

Go to the [Pub/Sub Console](https://console.cloud.google.com/cloudpubsub), select `Traffic` topic and click `Publish Message`.

Add following message.
```json
{
  "Location": "Pori",
  "Time": "2018-04-02T14:32:56+03:00",
  "Direction": 1,
  "VehicleType": "passenger_car",
  "Speed": 98
}
```

Verify that row is in Traffic table.

```sql
SELECT * FROM `Workshop.Traffic`
WHERE Time = '2018-04-02T14:32:56+03:00'
```

### Links

[Writing Cloud Functions](https://cloud.google.com/functions/docs/writing/)

[BigQuery NodeJS](https://cloud.google.com/nodejs/docs/reference/bigquery/1.0.x/BigQuery)

## Extra: Dataflow

[Extra: Dataflow exercise](traffic_dataflow.md)

## Cleanup Resources

### Step 1 - Cloud Functions

Go to the [Cloud Functions Console](https://console.cloud.google.com/functions/)
* Select `Traffic-Function`
* Click Delete button

### Step 2 - Pub/Sub

Go to the [Pub/Sub Console](https://console.cloud.google.com/cloudpubsub)

* Select `Traffic`
* Click Delete button

### Step 3 - BigQuery

Go to the [BigQuery Console](https://bigquery.cloud.google.com)

* Click dropdown menu from `Workshop` dataset
* Click Delete dataset
* Write dataset name to confirm delete
* Click Ok
