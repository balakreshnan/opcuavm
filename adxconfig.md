# Industrial IoT with OPC UA data

## Configure Industrial IoT with OPC UA data stored in Azure data explorer

## Prerequisites

- Azure Account
- Azure IoT Hub
- Azure Container Registry
- Azure Storage
- Azure Data Explorer

## Code

- Create a Azure data explorer cluster
- Create a database with 31 days cache and 365 days storage
- Create a table

```
.create table opcuaincoming (data:dynamic)
```

- Now create a mapping

```
.create table opcuaincoming ingestion json mapping 'opcuaincomingMapping' '[{"column":"data","path":"$","datatype":"dynamic"}]'
```

- Create a function

```
.create-or-alter function telemetry(){
opcuaincoming
| project  ContentMask = tostring(data.ContentMask),
NodeId = tostring(data.NodeId), EndpointUrl = tostring(data.EndpointUrl), 
DisplayName = tostring(data.DisplayName),  Value = tostring(data.Value.Body),
ValueType = tostring(data.Value.Type), 
SourceTimestamp= todatetime(data.SourceTimestamp),
ServerTimestamp= todatetime(data.ServerTimestamp),
Status = tostring(data.Status), Timestamp = todatetime(data.Timestamp)
}
```

- Now create the actual table to expand

```
.set devicetelemetry <|
    telemetry()
    | limit 0
```

- Create the trigger to map to actual columns

```
//  Create an update policy to transfer landing to landingTransformed
.alter table devicetelemetry policy update
@'[{"IsEnabled": true, "Source": "opcuaincoming", "Query": "telemetry", "IsTransactional": true, "PropagateIngestionProperties": false}]'
```

- Now lets set the expiration to 10 minutes

```
.alter table opcuaincoming policy retention "{'SoftDeletePeriod': '00:10:00', 'Recoverability':'Enabled'}"
```

- Query the data and make sure data flows
- Wait atleast 5 mins for data to show up

```
devicetelemetry
| count
```

```
devicetelemetry
| limit 1000
| order  by SourceTimestamp desc 
```

```
.create-or-alter function telemetryaggr(){
opcuaincoming
| project NodeId = tostring(data.NodeId), 
AggrTime = todatetime(data.Time), Count=toint(data.Count)
| where isnotempty(Count)
}
```

```
.set devicetelemetryaggr <|
    telemetryaggr()
    | limit 0
```

```
//  Create an update policy to transfer landing to landingTransformed
.alter table devicetelemetryaggr policy update
@'[{"IsEnabled": true, "Source": "opcuaincoming", "Query": "telemetryaggr", "IsTransactional": true, "PropagateIngestionProperties": false}]'
```

```
devicetelemetryaggr
| limit 100
```