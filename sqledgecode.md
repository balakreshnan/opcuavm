# Azure SQL Edge Streaming Code

## Streaming code

## Code

- Aggregate data
- Send all OPC UA data to SQL Edge
- DeviceData will store all the data
- deviceaggr will store 1 minute summation

- First container login

```
az acr login --name opcuacontainer
```

- Copy the sql files to edge devices
- Then copy to AzureSQLEdge container
- Install AzureSQLEdge module from portal

```
sudo docker cp dbinit.sql AzureSQLEdge:/usr/share
sudo docker cp streaminit.sql AzureSQLEdge:/usr/share
sudo docker cp resetdbforexport.sql AzureSQLEdge:/usr/share
```

- Now log into Docker AzureSQledge container

```
sudo docker exec -it AzureSQLEdge "bash"
```

- Now copy the above files to container

```
/opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P 'password' -i /usr/share/dbinit.sql
/opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P 'password' -i /usr/share/streaminit.sql
```

- Commands to clear the stale docker images or cache

```
sudo iotedge system stop
sudo docker rm -f $(sudo docker ps -aq)
sudo docker rmi -f $(sudo docker image ls -aq)
sudo iotedge system restart
```

- Routes used for edge

```
Routes:

cloud : FROM /messages/modules/opcpublisher/* INTO $upstream

opc : FROM /messages/modules/opcpublisher/* INTO BrokeredEndpoint("/modules/AzureSQLEdge/inputs/OPCUAData")

asatohub: FROM /messages/modules/saedge/* INTO $upstream

asatosql: FROM /messages/modules/saedge/* INTO BrokeredEndpoint("/modules/AzureSQLEdge/inputs/asaData")
```

- Log into SQL Edge

```
sudo docker exec -it AzureSQLEdge "bash"
/opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P 'password'
```

- create a table to store aggregation

```
Create Table dbo.deviceaggr
(
[NodeId] NVARCHAR(255) NULL,
[Time] DATETIME2,
[Count] FLOAT
)
GO
```

- Stop stream code

```
EXEC sys.sp_start_streaming_job @name=N'dashboardsStreamFromHubIntoTable'

EXEC sys.sp_stop_streaming_job @name=N'dashboardsStreamFromHubIntoTable'

EXEC sys.sp_drop_streaming_job @name=N'dashboardsStreamFromHubIntoTable'
```

- Streaming code to save actual devicedata, deviceaggr and output to iot edge
- Make sure the new line betweeen two sql queries and last ;
- ' should be in next line

- Create the data sources
- Create new input stream for all 3 queries

```
CREATE EXTERNAL STREAM dashboardsOPCUAInputStream WITH ( DATA_SOURCE = dashboardsInput, FILE_FORMAT = dashboardsInputFileFormat, LOCATION = N'OPCUAData', INPUT_OPTIONS = N'', OUTPUT_OPTIONS = N'')
CREATE EXTERNAL STREAM dashboardsOPCUAInputStream1 WITH ( DATA_SOURCE = dashboardsInput, FILE_FORMAT = dashboardsInputFileFormat, LOCATION = N'OPCUAData', INPUT_OPTIONS = N'', OUTPUT_OPTIONS = N'')
CREATE EXTERNAL STREAM dashboardsOPCUAInputStream2 WITH ( DATA_SOURCE = dashboardsInput, FILE_FORMAT = dashboardsInputFileFormat, LOCATION = N'OPCUAData', INPUT_OPTIONS = N'', OUTPUT_OPTIONS = N'')
```

```
CREATE EXTERNAL DATA SOURCE dashboardsOutput WITH (LOCATION = 'edgehub://')

CREATE EXTERNAL STREAM DeviceAggrTable WITH (DATA_SOURCE = telemetryDbServer,LOCATION = N'telemetry.dbo.deviceaggr',INPUT_OPTIONS = N'',OUTPUT_OPTIONS = N'')
CREATE EXTERNAL STREAM edgeHubOut WITH ( DATA_SOURCE = dashboardsOutput, FILE_FORMAT = dashboardsInputFileFormat, LOCATION = 'edgehub', OUTPUT_OPTIONS = 'REJECT_TYPE: Drop' );
```

- Streaming configuration

```
EXEC sys.sp_create_streaming_job @name=N'dashboardsStreamFromHubIntoTable',
@statement= N' Select ContentMask, NodeId, ServerTimestamp, SourceTimestamp, StatusCode, Status, ApplicationUri, Timestamp, Value.Type as [ValueType], Value.Body as Value, substring(NodeId,regexmatch(NodeId,''=(?:.(?!=))+$'')+1,len(NodeId)-regexmatch(NodeId,''=(?:.(?!=))+$'')) as DataPoint, substring(NodeId,1, regexmatch(NodeId,''\#(?:.(?!\#))+$'')-1) as Asset, SourceTimestamp as [time] into DeviceDataTable from dashboardsOPCUAInputStream 

SELECT NodeId, System.Timestamp() AS [Time], COUNT(*) AS [Count] INTO DeviceAggrTable FROM dashboardsOPCUAInputStream1 TIMESTAMP BY SourceTimestamp GROUP BY NodeId, TumblingWindow(second, 60)

SELECT NodeId, System.Timestamp() AS [Time], COUNT(*) AS [Count] INTO edgeHubOut FROM dashboardsOPCUAInputStream2 TIMESTAMP BY SourceTimestamp GROUP BY NodeId, TumblingWindow(second, 60);
'
go
```

- More to come ...