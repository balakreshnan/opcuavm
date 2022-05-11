# Industrial IoT Demo

This project is a simplification of the [IOT-Edge-For-IIOT](https://github.com/Azure-Samples/iot-edge-for-iiot) sample.  It aims to give a developer an understanding of how to leverage the OPCPublisher and Azure SQL Edge to create near real time Grafana dashboards at the Edge.  

## Solution Overview

![Industrial IoT Demo Architecture](/assets/architecture.png)
- opcsimulator is a NodeRed container that has two flows representing two OPCServers exposing container ports 54845 and 54855.  It exposes port 1880 so you can log into the NodeRed environment and explore the flows.
- opcpublisher is configured to read from opcsimulator and output data to edgeHub where it is routed to sqledge via the "opc" route and IoTHub via the "cloud" route.
- sqledge is an Azure SQL Edge container that contains a data instance for storing telemetry and a stream analytics job that reads in the data from the opcpublisher via the "opc" route.    It exposes port 1433 so you can connect with tools such as SQL Server Management Studio or Azure Data Studio.
- grafana is a deployment of the grafana container.  It connects to sqledge to read for dashboards.  It exposes port 3000 so you can connect via a web browser.
- iotedgemetricscollector is the IoT Edge Metrics Collector and sends data to a Log Analytics Workspace via https.

## Requirements

1. IoT Hub - [Instructions](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-create-through-portal?view=iotedge-2020-11#create-an-iot-hub)
2. Ubuntu VM with IoT Edge installed - [Instructions](https://docs.microsoft.com/azure/iot-edge/how-to-provision-single-device-linux-symmetric?view=iotedge-2020-11&tabs=azure-portal%2Cubuntu)
3. Azure Container Registry - [Instructions](https://docs.microsoft.com/Azure/container-registry/container-registry-get-started-portal#create-a-container-registry)
4. Log Analytics Workspace - [Instructions](https://docs.microsoft.com/en-us/azure/azure-monitor/logs/quick-create-workspace)
5. Visual Studio Code - [Download Page](https://code.visualstudio.com/) and [Azure IoT Tools Extension Page](https://marketplace.visualstudio.com/items?itemName=vsciot-vscode.azure-iot-tools)
6. SQL Server Management Studio [https://docs.microsoft.com/sql/ssms/download-sql-server-management-studio-ssms?view=sql-server-ver15]

## Deployment

1. Clone this repo locally.
2. Rename .envsample to .env and enter the required data points.
3. Optional: the sql edge sa credential password is set to 'Password_54321'.  You can do a search and replace across the files to change it.
4. From the terminal command line, connect to your azure container registry using `az acr login --name <registry name>`.
5. Right click on deployment.template.json and select 'Build and Push IoT Edge Solution'.  This will build containers for all the modules and push them to your azure container registry.
6. Right click on deployment.template.json and select 'IoT Edge Deployment Manifest'.  This will create a deployment file for the AMD64 platform in the config directory.
7. Right click on deployment.amd64.json and select 'Create Deployment for Single Device' and then select your iot edge.

## Evaluating Results

### IoT Edge

1. Log into your iot edge box.
1. View the running modules using `sudo iotedge list`
1. View the logs of each module using `sudo iotedge logs <module name>`

### OPC Simulator

1. Open inbound port 1880
2. Use a web browser to connect to http://<servername>:1880
3. Explore the NodeRed environment for how the simulator works.

### OPC Publisher

1. Open [publishedNodes.json](/modules/opcpublisher/publishedNodes.json)
2. Explore the configuration for how it reads the OPC UA data.

### SQL Edge

1. Open inbound port 1433
2. Using SQL Server Management Studio, connect to your VM and explore the tables and views.  Default credentials are 'sa' and 'Password_54321'
3. Open [streaminit.sql](/modules/sqledge/streaminit.sql) to see how the stream analytics job is setup and connects to iot edge to read the data.

### Grafana

1. Open inbound port 3000
2. Use a web browser to connect to http://<servername>:3000.  Default credentials are 'admin/admin'
3. Open the different dashboards to see the performance of the opc simulator.

### IoT Edge Metrics Collector

1. In the Azure Portal, navigate to your IoT Hub resource.
2. Select the 'Workbooks' blade under 'Monitoring'
3. Explore the different IoT Edge workbooks.

### Defender for IoT

1. In the Azure Portal, navigate to your IoT Hub resource.
2. Select the 'Overview' blade under 'Defender for IoT'.
3. Explore the different options and alerts.

## Next Steps

This sample created a single node IIOT deployment to provide a high level understanding of how to leverage OPC UA and offline dashboards at the edge.  Consider deploying the [IOT-Edge-For-IIOT](https://github.com/Azure-Samples/iot-edge-for-iiot) sample to learn how the IoT Edge nested edge architecture works in a Purdue Network, and how automation can be applied.