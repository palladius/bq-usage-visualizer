# bq-usage-visualizer
A solution for visualizing the query costs and usage over day, generated in BigQuery, for a Google Compute Platform project.

Using the following guide you will end up with a spreadsheet showing the BigQuery query costs over time and per user in near real-time.

This is acheived by plumbing together already existing Google technologies, i.e. you will (almost) not have to deploy code anywhere.

### Example real-time graphs
![Total usage last 30 days](https://raw.githubusercontent.com/fwallenius/bq-usage-visualizer/master/images/graph_month.png)
![Top users last 30 days](https://raw.githubusercontent.com/fwallenius/bq-usage-visualizer/master/images/graph_month2.png)
![Top users last 7 days](https://raw.githubusercontent.com/fwallenius/bq-usage-visualizer/master/images/graph_lastweek.png)

## High-level description

Google Cloud Platform (GCP) has audit logging available for most of the products. In this solution we setup streaming of the BigQuery (BQ) audit logs into, well, BigQuery. Every query run in BQ generates an audit log entry that mong other things states who ran the query and how many bytes that was processed. 

We then create a Google Apps spreadsheet that queries the audit logs to generate graphs and diagrams showing details over the last 30 days of usage for the given GCP project.

@TODO: Show example graph here

## Step by step guide

In the name of speed and efficiency most of the plumbing is done using the command line terminal. The full setup can be done in the Google Cloud Developer Console as well.

### Pre-conditions

* Create a new project or select an existing one. 
* Grab the Project ID and Project number from the Developer Console:
![Grab project ID and number](https://raw.githubusercontent.com/fwallenius/bq-usage-visualizer/master/images/project_id.png)

* Fire up a bash session and store them in two variables:
```bash
$ export PROJECT_ID=the-bigquery-project
$ export PROJECT_NUMBER=600424227998
```
* Then make sure your gcloud command defaults to this project:
```bash
$ gcloud config set project $PROJECT_ID
```

### Setup the logging and BigQuery tables

* First we need a dataset in BigQuery where the audit logs will be stored. Let’s call it “bqusage”. Create it by running:
```bash
$ bq mk bqusage
```

* Cloud Logging needs to be granted permission to stream the data into our dataset. This can only be done from the UI: 

  * Open https://bigquery.cloud.google.com/
  * Find your new dataset “bqusage” in the list on the left and select “Share dataset”:
![Select Share dataset](https://raw.githubusercontent.com/fwallenius/bq-usage-visualizer/master/images/share_dataset.png)
  * In the pop-up, change left dropdown to “Group by e-mail”, enter cloud-logs@google.com as email and change the right dropdown to Edit access.
![Add the Logs users](https://raw.githubusercontent.com/fwallenius/bq-usage-visualizer/master/images/add_logs_user.png)
  * Click Add and then Save changes.

* Next we’ll activate the usage logging and make sure it’s stored in the new dataset:
```bash
$ gcloud beta logging sinks create bq-usage bigquery.googleapis.com/projects/$PROJECT_NUMBER/datasets/bqusage --log-service bigquery.googleapis.com
```

* Before creating a graph we need to make sure we have some usage logged by running a query on one of the public data sets. Run the following command to see which five locations in New York had the most Yellow Cab pickups during December 2015:
```bash
$ bq query '
SELECT
  COUNT (*) AS Pickups,
  ROUND(pickup_latitude, 4) AS Latitude,
  ROUND(pickup_longitude, 4) AS Longitude
FROM
  [nyc-tlc:yellow.trips_2015_12]
WHERE
  pickup_latitude != 0.0
GROUP BY
  Longitude,
  Latitude
ORDER BY
  Pickups DESC
LIMIT
  5’
```

* Next we validate that the usage logs are being streamed into our data set:
```bash
$ bq ls bqusage
```
The output should be similar to this:
```bash

                    tableId                       Type
------------------------------------------------ -------
 cloudaudit_googleapis_com_data_access_20160414   TABLE
```

* The audit logging dataset is a bit clunky to query. We’ll create a view to simplify it:
```bash
$ bq mk --view='
SELECT
  ts,
  total_bytes,
  user,
  projectId
FROM (
  SELECT
    metadata.timestamp AS ts,
    protoPayload.serviceData.jobCompletedEvent.job.jobStatistics.totalBilledBytes AS total_bytes,
    protoPayload.authenticationInfo.principalEmail AS user,
    metadata.projectId AS projectId
  FROM
    TABLE_DATE_RANGE([bqusage.cloudaudit_googleapis_com_data_access_], DATE_ADD(CURRENT_TIMESTAMP(), -30, "DAY"), CURRENT_TIMESTAMP())
  WHERE
    protoPayload.serviceData.jobCompletedEvent.eventName = "query_job_completed")
WHERE
  total_bytes > 0
ORDER BY
  ts DESC' bqusage.usage_simplified
```

### Visualizing the data using Apps Spreadsheet

Now we have all the data needed. Time to create the dashboard. Google Apps Spreadsheet has a BigQuery connector and can transform the data into graphs.

* Start by opening this template https://docs.google.com/spreadsheets/d/1pEDj-XFhpb6CkYU-bws0_nFOStWEjwasIcjvIOwIT2Q/edit?usp=sharing

* Make a copy of the temple owned by yourself.

* Open the "data" sheet and change the project ID cell to your project.

* Access the script editor by selecting Tools / Script Editor..

* Associate the spreadsheet with your GCP project by selecting Resources / Develops Console Project.. In the pop-up enter your project number and click "Set Project"

* Run the script by clicking the play button. On first run you will have to approve the spreadsheet's access to your BigQuery dataset.
















