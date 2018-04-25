## Traffic Dataflow

### Pre-Requirements
- [Google Cloud SDK](https://cloud.google.com/sdk)
- Python 3.5
- Java 8
- Following APIs needs to be enabled in [APIs & Services](https://console.cloud.google.com/apis)
  - Dataflow API
  - Google Cloud Storage

### Step 1 - Create Statistics Table

Go to the [BigQuery Console](https://bigquery.cloud.google.com)

| Parameter    | Value               |
| ------------ | ------------------- |
| Source Data  | Create empty table  |
| Table name   | `TrafficStatistics` |

| Schema Field | Type        | Mode       |
| ------------ | ----------- | ---------- |
| Location     | `STRING`    | `NULLABLE` |
| Time         | `TIMESTAMP` | `NULLABLE` |
| AverageSpeed | `FLOAT`     | `NULLABLE` |
| MinSpeed     | `INTEGER`   | `NULLABLE` |
| MaxSpeed     | `INTEGER`   | `NULLABLE` |
| Count        | `INTEGER`   | `NULLABLE` |

### Step 2 - Create Bucket for Dataflow

Go to the [Storage Console](https://console.cloud.google.com/storage)

- Click Create Bucket

| Parameter | Value    |
| ----------| -------- |
| Name      | `my-project-id-dataflow` !! Replace `my-project-id` with your Google project id|
| Location  | `Europe` |

### Step 3 - Create Java Application for Dataflow

This dataflow reads Traffic message from PubSub and writes those to Traffic table in BigQuery. It also calculates speed statistics (average, min, max and count) and writes those to TrafficStatistics table. Statistics are calculated every minute for each location.

`build.gradle`

```gradle
apply plugin: 'java'
apply plugin: 'eclipse'
apply plugin: 'application'

repositories {
    jcenter()
}

dependencies {
    compile group: 'com.google.cloud.dataflow', name: 'google-cloud-dataflow-java-sdk-all', version: '2.2.0'
    compile group: 'org.apache.commons', name: 'commons-math3', version: '3.6.1'
    compile group: 'org.slf4j', name: 'slf4j-simple', version: '1.7.25'
}

mainClassName = 'workshop.traffic.Dataflow'
```

`workshop/traffic/Dataflow.java`

```java
package workshop.traffic;

import java.io.Serializable;

import org.apache.beam.runners.dataflow.DataflowRunner;
import org.apache.beam.runners.dataflow.options.DataflowPipelineOptions;
import org.apache.beam.sdk.Pipeline;
import org.apache.beam.sdk.io.gcp.bigquery.BigQueryIO;
import org.apache.beam.sdk.io.gcp.pubsub.PubsubIO;
import org.apache.beam.sdk.options.PipelineOptionsFactory;
import org.apache.beam.sdk.transforms.*;
import org.apache.beam.sdk.transforms.windowing.*;
import org.apache.beam.sdk.values.KV;
import org.apache.beam.sdk.values.PCollection;
import org.apache.commons.math3.stat.descriptive.SummaryStatistics;
import org.joda.time.Duration;
import org.joda.time.Instant;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.google.api.services.bigquery.model.TableReference;
import com.google.api.services.bigquery.model.TableRow;

public class Dataflow implements Serializable
{
  protected static final Logger LOG = LoggerFactory.getLogger( Dataflow.class );

  private static final String PROJECT_ID = "my-project-id"; // Put your Google project id to here

  private static final String APP_NAME = "Traffic Dataflow";
  private static final String JOB_NAME = "traffic-dataflow";

  private static final String MACHINE_TYPE = "n1-standard-1";

  private static final String DATASET_ID = "Workshop";
  private static final String TABLE_ID_TRAFFIC = "Traffic";
  private static final String TABLE_ID_TRAFFIC_STATISTICS = "TrafficStatistics";

  private static final String COLUMN_TIME = "Time";
  private static final String COLUMN_LOCATION = "Location";
  private static final String COLUMN_SPEED = "Speed";
  private static final String COLUMN_AVERAGE_SPEED = "AverageSpeed";
  private static final String COLUMN_MIN_SPEED = "MinSpeed";
  private static final String COLUMN_MAX_SPEED = "MaxSpeed";
  private static final String COLUMN_COUNT = "Count";

  private static final String PUBSUB_TOPIC_NAME = "Traffic";
  private static final String TOPIC_ID = "projects/" + PROJECT_ID + "/topics/" + PUBSUB_TOPIC_NAME;

  private static final String DATAFLOW_BUCKET = PROJECT_ID + "-dataflow";
  private static final String STATING_LOCATION = "gs://" + DATAFLOW_BUCKET + "/staging";
  private static final String TEMP_LOCATION = "gs://" + DATAFLOW_BUCKET + "/temp";

  protected static final ObjectMapper MAPPER = new ObjectMapper( );

  public Dataflow( )
  {
  }

  public void run( )
  {
    DataflowPipelineOptions options = PipelineOptionsFactory.as( DataflowPipelineOptions.class );
    options.setProject( PROJECT_ID );
    options.setAppName( APP_NAME );
    options.setJobName( JOB_NAME );
    options.setStagingLocation( STATING_LOCATION );
    options.setTempLocation( TEMP_LOCATION );
    options.setWorkerMachineType( MACHINE_TYPE );
    options.setNumWorkers( 1 );
    options.setMaxNumWorkers( 1 );
    options.setRunner( DataflowRunner.class );

    Pipeline pipeline = Pipeline.create( options );

    PCollection<TableRow> rawDataRows = pipeline
        .apply( "Read Traffic Data", PubsubIO.readStrings( ).fromTopic( TOPIC_ID ) )//
        .apply( "Parse Traffic Data", ParDo.of( new DoFn<String, TableRow>( ) {
          @ProcessElement
          public void processElement( ProcessContext c )
          {
            try
            {
              TableRow tableRow = MAPPER.readValue( c.element( ), TableRow.class );
              Instant timeStamp = Instant.parse( (String)tableRow.get( COLUMN_TIME ) );
              c.outputWithTimestamp( tableRow, timeStamp );
            }
            catch( Exception e )
            {
              LOG.error( "Ignore bad data", e );
            }
          }
        } ) );

    TableReference trafficTable = new TableReference( );
    trafficTable.setProjectId( PROJECT_ID );
    trafficTable.setDatasetId( DATASET_ID );
    trafficTable.setTableId( TABLE_ID_TRAFFIC );

    rawDataRows.apply( "Write Raw Data",
        BigQueryIO.writeTableRows( ).to( trafficTable )
            .withCreateDisposition( BigQueryIO.Write.CreateDisposition.CREATE_NEVER )
            .withWriteDisposition( BigQueryIO.Write.WriteDisposition.WRITE_APPEND ) );

    PCollection<TableRow> statisticsRows = rawDataRows //
        .apply( "Fixed Window 1 min",
            Window.<TableRow> into( FixedWindows.of( Duration.standardMinutes( 1 ) ) )
                .triggering( AfterWatermark.pastEndOfWindow( ) )
                .withAllowedLateness( Duration.standardSeconds( 5 ) ).accumulatingFiredPanes( ) )
        .apply( "Extract Location", ParDo.of( new DoFn<TableRow, KV<String, TableRow>>( ) {
          @ProcessElement
          public void processElement( ProcessContext c )
          {
            TableRow tableRow = c.element( );
            String location = (String)tableRow.get( COLUMN_LOCATION );
            c.output( KV.of( location, tableRow ) );
          }
        } ) ) //
        .apply( "Group By Location", GroupByKey.create( ) ) //
        .apply( "Calculate Statistics",
            ParDo.of( new DoFn<KV<String, Iterable<TableRow>>, TableRow>( ) {
              @ProcessElement
              public void processElement( ProcessContext c, IntervalWindow window )
              {
                String location = c.element( ).getKey( );
                LOG.info( "Calculate statistics for " + location );
                TableRow outputTableRow = new TableRow( );
                outputTableRow.set( COLUMN_LOCATION, location );
                outputTableRow.set( COLUMN_TIME, window.start( ).toString( ) );
                SummaryStatistics statistics = new SummaryStatistics( );
                for( TableRow tableRow : c.element( ).getValue( ) )
                {
                  statistics.addValue( ( (Integer)tableRow.get( COLUMN_SPEED ) ).doubleValue( ) );
                }
                outputTableRow.set( COLUMN_AVERAGE_SPEED, Double.valueOf( statistics.getMean( ) ) );
                outputTableRow.set( COLUMN_MIN_SPEED, Integer.valueOf( (int)statistics.getMax( ) ) );
                outputTableRow.set( COLUMN_MAX_SPEED, Integer.valueOf( (int)statistics.getMin( ) ) );
                outputTableRow.set( COLUMN_COUNT, Long.valueOf( statistics.getN( ) ) );
                c.output( outputTableRow );
              }
            } ) );

    TableReference trafficStatisticsTable = new TableReference( );
    trafficStatisticsTable.setProjectId( PROJECT_ID );
    trafficStatisticsTable.setDatasetId( DATASET_ID );
    trafficStatisticsTable.setTableId( TABLE_ID_TRAFFIC_STATISTICS );

    statisticsRows.apply( "Write Statistics",
        BigQueryIO.writeTableRows( ).to( trafficStatisticsTable )
            .withCreateDisposition( BigQueryIO.Write.CreateDisposition.CREATE_NEVER )
            .withWriteDisposition( BigQueryIO.Write.WriteDisposition.WRITE_APPEND ) );

    pipeline.run( );
  }

  public static void main( String[] args )
  {
    new Dataflow( ).run( );
  }
}
```

### Step 4 - Create Python Application for PubSub

This application sends random generated Traffic messages to PubSub.

```python
import json
import datetime
import time
import pytz
from random import randint
from google.cloud import pubsub

project = 'my-project-id' # Put your Google project id to here
topic = 'Traffic'

locations = ['Pori', 'Nakkila', 'Vihti', 'Karkkila']
directions = [1, 2]
vehicleTypes = ['passenger_car', 'truck', 'bus']
speedMinMax = [70, 130]

local_tz = pytz.timezone("Europe/Helsinki")


def get_random(array):
    return array[randint(0, len(array) - 1)]


def get_time_with_timezone():
    return local_tz.localize(datetime.datetime.now()).isoformat()


def publish(sleepTimeInSec):
    publisher = pubsub.PublisherClient()
    pubSubTopic = 'projects/{project}/topics/{topic}'.format(
        project=project, topic=topic)
    while (True):
        data = {
            "Location": get_random(locations),
            "Time": get_time_with_timezone(),
            "Direction": get_random(directions),
            "VehicleType": get_random(vehicleTypes),
            "Speed": randint(speedMinMax[0], speedMinMax[1])
        }
        jsonData = json.dumps(data)
        print(jsonData)
        publisher.publish(pubSubTopic, bytes(jsonData, 'UTF-8'))
        time.sleep(sleepTimeInSec)


publish(2)
```

### Step 5 - Dataflow Console

Go to the [Dataflow Console](https://console.cloud.google.com/dataflow)

Open `traffic-dataflow` dataflow.


