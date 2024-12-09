### BigQuery & PostgreSQL FDW

> Disclaimer + Note : This documentation Is Generic & On Microsoft Developer APIs Documentation

> Reference : https://learn.microsoft.com/en-us/azure/virtual-machines/windows/scheduled-events

--

#### What Is The Idea Here ?

- Step-1 : Source Table : `my-project.my_data_set.my_custom_table`
- Step-2 : Create View : `my-project.my_data_set.view_my_custom_table`
- Step-3 : Install & Setup Multicorn PostgreSQL Extension (FDW)
- Step-4 : Create PostgreSQL FDW Table `my_microsoft_events_table`
- Step-5 : Create PostgreSQL Materialized View `my_microsoft_events_table_mv`

> Source Table

`SELECT * FROM `my-project.my_data_set.my_custom_table` LIMIT 1;`

--

> Columns

- `timestamp`
- `event_data`

--

> Sample Row Value

Column : `timestamp`

`2024-12-04 18:01:17.545601 UTC`

Column : `event_data`

```json
{
    "data":
    {
        "apiVersion": "2020-03-01",
        "operationalInfo":
        {
            "resourceEventTime": "2024-12-04T18:01:13.2675752Z"
        },
        "resourceInfo":
        {
            "id": "/subscriptions/62d72a8a-5a42-4a28-a1cc-70a836ae7955/resourcegroups/my-resource-group-001/providers/microsoft.compute/virtualmachines/my_vm_001/providers/microsoft.maintenance/scheduledevents/743c15d8-a994-46fc-93af-b84ee33f9a14",
            "location": "westus2",
            "name": "743c15d8-a994-46fc-93af-b84ee33f9a14",
            "properties":
            {
                "Description": "Virtual machine is undergoing maintenance",
                "DurationInSeconds": -1,
                "EventId": "743c15d8-a994-46fc-93af-b84ee33f9a14",
                "EventSource": "Platform",
                "EventStatus": "Completed",
                "EventType": "Freeze",
                "NotBefore": "",
                "ResourceType": "VirtualMachine",
                "Resources":
                [
                    "my_vm_001"
                ]
            },
            "tags":
            {},
            "type": "microsoft.maintenance/scheduledevents"
        }
    },
    "dataVersion": "1",
    "eventTime": "2024-12-04T18:01:13.2675752Z",
    "eventType": "Microsoft.ResourceNotifications.MaintenanceResources.ScheduledEventEmitted",
    "id": "6e9f8439-8bf5-48f1-805d-ad8ab7cd1951",
    "metadata":
    {
        "customRunIdentifier": "fNkLgg0G",
        "origin": "mock",
        "region": "my_region_01",
        "totalCount": 270,
        "uniqueRunIdentifier": "STuoMlU5"
    },
    "metadataVersion": "1",
    "received_timings":
    {
        "received_epoch_ms": 1733335276364,
        "received_time": 1733335276,
        "received_time_pst": "2024-12-04 10:01:16.364146 PST",
        "received_time_utc": "2024-12-04 18:01:16.364142 UTC"
    },
    "sent_timings":
    {
        "sent_epoch_ms": 1733335277545,
        "sent_time": 1733335277,
        "sent_time_pst": "2024-12-04 10:01:17.5451 PST",
        "sent_time_utc": "2024-12-04 18:01:17.545096 UTC"
    },
    "subject": "/subscriptions/62d72a8a-5a42-4a28-a1cc-70a836ae7955/resourcegroups/my-resource-group-001/providers/microsoft.compute/virtualmachines/my_vm_001",
    "tags":
    {
        "vm_name": "my_vm_001",
        "vm_resource_group": "my-resource-group-001"
    },
    "topic": "/subscriptions/62d72a8a-5a42-4a28-a1cc-70a836ae7955"
}
```

--

#### Create BQ View From Table

```sql
CREATE OR REPLACE VIEW `my-project.my_data_set.view_my_custom_table` AS
WITH unnested_resources AS (
    SELECT
        timestamp,
        event_data,
        JSON_EXTRACT_SCALAR(resource) as resource  -- Extract scalar value from JSON
    FROM `my-project.my_data_set.my_custom_table`,
    UNNEST(JSON_EXTRACT_ARRAY(event_data, "$.data.resourceInfo.properties.Resources")) as resource
    WHERE timestamp >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 10 DAY)
    AND timestamp <= CURRENT_TIMESTAMP()
),
parsed_data AS (
    SELECT
        timestamp,
        event_data,
        resource,
        CASE 
            WHEN JSON_EXTRACT_SCALAR(event_data, "$.received_timings.received_time_pst") IS NULL THEN NULL
            WHEN REGEXP_CONTAINS(JSON_EXTRACT_SCALAR(event_data, "$.received_timings.received_time_pst"), r' PST$') 
            THEN REGEXP_REPLACE(JSON_EXTRACT_SCALAR(event_data, "$.received_timings.received_time_pst"), r' PST$', '')
            ELSE JSON_EXTRACT_SCALAR(event_data, "$.received_timings.received_time_pst")
        END as clean_received_pst,
        REGEXP_REPLACE(
            CASE 
                WHEN JSON_EXTRACT_SCALAR(event_data, "$.received_timings.received_time_utc") IS NULL THEN NULL
                WHEN REGEXP_CONTAINS(JSON_EXTRACT_SCALAR(event_data, "$.received_timings.received_time_utc"), r' UTC$') 
                THEN REGEXP_REPLACE(JSON_EXTRACT_SCALAR(event_data, "$.received_timings.received_time_utc"), r' UTC$', '')
                ELSE JSON_EXTRACT_SCALAR(event_data, "$.received_timings.received_time_utc")
            END,
            r'(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}\.\d{6})\d*\s*',
            r'\1'
        ) as clean_received_utc,
        CASE 
            WHEN JSON_EXTRACT_SCALAR(event_data, "$.sent_timings.sent_time_pst") IS NULL THEN NULL
            WHEN REGEXP_CONTAINS(JSON_EXTRACT_SCALAR(event_data, "$.sent_timings.sent_time_pst"), r' PST$') 
            THEN REGEXP_REPLACE(JSON_EXTRACT_SCALAR(event_data, "$.sent_timings.sent_time_pst"), r' PST$', '')
            ELSE JSON_EXTRACT_SCALAR(event_data, "$.sent_timings.sent_time_pst")
        END as clean_sent_pst,
        CASE 
            WHEN JSON_EXTRACT_SCALAR(event_data, "$.sent_timings.sent_time_utc") IS NULL THEN NULL
            WHEN REGEXP_CONTAINS(JSON_EXTRACT_SCALAR(event_data, "$.sent_timings.sent_time_utc"), r' UTC$') 
            THEN REGEXP_REPLACE(JSON_EXTRACT_SCALAR(event_data, "$.sent_timings.sent_time_utc"), r' UTC$', '')
            ELSE JSON_EXTRACT_SCALAR(event_data, "$.sent_timings.sent_time_utc")
        END as clean_sent_utc,
        REGEXP_REPLACE(
            CASE 
                WHEN JSON_EXTRACT_SCALAR(event_data, "$.eventTime") IS NULL THEN NULL
                WHEN REGEXP_CONTAINS(JSON_EXTRACT_SCALAR(event_data, "$.eventTime"), r'Z$') 
                THEN REGEXP_REPLACE(JSON_EXTRACT_SCALAR(event_data, "$.eventTime"), r'T|Z$', ' ')
                ELSE REGEXP_REPLACE(JSON_EXTRACT_SCALAR(event_data, "$.eventTime"), r'T', ' ')
            END,
            r'(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}\.\d{6})\d*\s*',
            r'\1'
        ) as clean_event_time
    FROM unnested_resources
)
SELECT
    FORMAT_TIMESTAMP('%Y-%m-%d %H:%M:%S.%E6S', timestamp) as time_stamp,
    JSON_EXTRACT_SCALAR(event_data, "$.data.resourceInfo.properties.EventId") as event_id,
    JSON_EXTRACT_SCALAR(event_data, "$.data.resourceInfo.properties.EventSource") as event_source,
    JSON_EXTRACT_SCALAR(event_data, "$.data.resourceInfo.properties.EventStatus") as event_status,
    JSON_EXTRACT_SCALAR(event_data, "$.data.resourceInfo.properties.EventType") as event_type,
    JSON_EXTRACT_SCALAR(event_data, "$.data.resourceInfo.properties.NotBefore") as not_before,
    CAST(JSON_EXTRACT_SCALAR(event_data, "$.received_timings.received_epoch_ms") AS INT64) as received_epoch_ms,
    CAST(JSON_EXTRACT_SCALAR(event_data, "$.received_timings.received_time") AS INT64) as received_time,
    clean_received_pst as received_time_pst,
    clean_received_utc as received_time_utc,
    CAST(JSON_EXTRACT_SCALAR(event_data, "$.sent_timings.sent_epoch_ms") AS INT64) as sent_epoch_ms,
    CAST(JSON_EXTRACT_SCALAR(event_data, "$.sent_timings.sent_time") AS INT64) as sent_time,
    clean_sent_pst as sent_time_pst,
    clean_sent_utc as sent_time_utc,
    JSON_EXTRACT_SCALAR(event_data, "$.subject") as subject,
    JSON_EXTRACT_SCALAR(event_data, "$.id") as id,
    clean_event_time as event_time,
    JSON_EXTRACT_SCALAR(event_data, "$.topic") as topic,
    CAST(JSON_EXTRACT_SCALAR(event_data, "$.sent_timings.sent_epoch_ms") AS INT64) - 
    CAST(JSON_EXTRACT_SCALAR(event_data, "$.received_timings.received_epoch_ms") AS INT64) as processing_time_ms,
    TIMESTAMP_DIFF(
        PARSE_TIMESTAMP('%Y-%m-%d %H:%M:%E6S', clean_received_utc),
        PARSE_TIMESTAMP('%Y-%m-%d %H:%M:%E6S', clean_event_time),
        SECOND
    ) as msft_delay_secs,
    resource
FROM parsed_data;
```

> Query View

```sql
select * from `my-project.my_data_set.view_my_custom_table`;
```

--

#### Multicorn For PostgreSQL

- Requirements
  - PostgreSQL >= 9.5 up to 14
  - Python >= 3.4

--

#### Install required packages

```shell
apt update
apt install -y postgresql-server-dev-14 python3-setuptools python3-dev make gcc git
```

--

#### Multicorn FDW (Foreign Data Wrapper) Extension For PostgreSQL

```
- Install Multicorn
- pgsql-io/multicorn2 is a fork of Segfault-Inc/Multicorn that adds support for PostgreSQL 13/14.
- Alternatively, up to PostgreSQL 12, you can use gabfl/Multicorn that adds better support for Python3.
- You may also choose to build against the original project instead.
```

```bash
git clone https://github.com/pgsql-io/multicorn2.git Multicorn && cd Multicorn

make && make install
```

>  Install bigquery_fdw

```bash
pip3 install bigquery-fdw
```

--

```bash
GOOGLE_APPLICATION_CREDENTIALS = '/path/to/key.json'
```

--

```sql
CREATE EXTENSION multicorn;

CREATE SERVER bigquery_srv FOREIGN DATA WRAPPER multicorn OPTIONS (
    wrapper 'bigquery_fdw.fdw.ConstantForeignDataWrapper'
);

CREATE FOREIGN TABLE my_bigquery_table (
    column1 text,
    column2 bigint
) SERVER bigquery_srv
OPTIONS (
    fdw_dataset  'my_dataset',
    fdw_table 'my_table'
);
```

--

> If required*

```sql
DROP FOREIGN TABLE my_microsoft_events_table;
```

> Create Foreign Table

```sql
CREATE FOREIGN TABLE my_microsoft_events_table (
    time_stamp TEXT,
    event_id TEXT,
    event_source TEXT,
    event_status TEXT,
    event_type TEXT,
    not_before TEXT,
    received_epoch_ms BIGINT,
    received_time BIGINT,
    received_time_pst TEXT,
    received_time_utc TEXT, 
    sent_epoch_ms BIGINT,
    sent_time BIGINT,
    sent_time_pst TEXT,
    sent_time_utc TEXT,
    subject TEXT,
    id TEXT,
    event_time TEXT,
    topic TEXT,
    processing_time_ms BIGINT,
    msft_delay_secs BIGINT,
    resource TEXT
)
SERVER bigquery_srv
OPTIONS (
    fdw_dataset 'my_data_set',
    fdw_table 'view_my_custom_table'
);
```

--

> If required*

```
DROP MATERIALIZED VIEW IF EXISTS microsoft_events_table_mv;
```

> Create

```sql
CREATE MATERIALIZED VIEW my_microsoft_events_table_mv 
WITH (fillfactor = 90) -- Helps with concurrent refresh performance
AS 
SELECT 
    time_stamp,
    event_id,
    event_source,
    event_status,
    event_type,
    not_before,
    received_epoch_ms,
    received_time,
    received_time_pst,
    received_time_utc,
    sent_epoch_ms,
    sent_time,
    sent_time_pst,
    sent_time_utc,
    subject,
    id,
    event_time,
    topic,
    processing_time_ms,
    msft_delay_secs,
    resource
FROM my_microsoft_events_table
WITH NO DATA; -- Initially create empty, then refresh
```

--

> This combination will most likely gaurantee unique index

```sql
CREATE UNIQUE INDEX my_microsoft_events_table_mv_unique_idx ON my_microsoft_events_table_mv  (id, received_epoch_ms, sent_epoch_ms);
```

> First time

```sql
REFRESH MATERIALIZED VIEW my_microsoft_events_table_mv;
```

> Consequently

```
REFRESH MATERIALIZED VIEW CONCURRENTLY my_microsoft_events_table_mv;
```

> Cron Job

```bash
#!/bin/bash
# refresh_mv.sh

# PostgreSQL connection details
export PGHOST='your_host'
export PGDATABASE='your_database'
export PGUSER='your_user'
export PGPASSWORD='your_password'

# Log file
LOG_FILE="/path/to/refresh_mv.log"

# Refresh command with logging
psql -c "REFRESH MATERIALIZED VIEW CONCURRENTLY microsoft_events_table_mv;" >> "$LOG_FILE" 2>&1

# Add error handling
if [ $? -ne 0 ]; then
    echo "Error refreshing materialized view at $(date)" >> "$LOG_FILE"
    exit 1
else
    echo "Successfully refreshed materialized view at $(date)" >> "$LOG_FILE"
fi
```

--

```bash
chmod +x refresh_mv.sh
```

--

```bash
# Edit crontab
crontab -e
```

```bash
# Add this line to refresh every hour
0 * * * * /path/to/refresh_mv.sh
```