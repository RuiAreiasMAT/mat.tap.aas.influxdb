# ![logo](/docs/branding.bmp) Telemetry Analytics Data Persistence

Telemetry data and metadata about sessions are saved to various storage implementations such as SqlRace and InfluxDb and relational databases. This article discusses saving telemetry data to InfluxDb as time-series data and metadata to MSSQL Server.

## InfluxDb installation
**We strongly recommend deploying InfluxDb writer and InfluxDb on one Linux VM. InfluxDb performs better on Linux and by deploying it on one machine together with service you reduce unnecessary network bandwidth between those services.**

Download and install InfluxDb following instructions [here](https://portal.influxdata.com/downloads).

**Change index-version in influx.config (in linux is located in /etc/influxdb/influxdb.config)**:

```
# The type of shard index to use for new shards.  The default is an in-memory index that is
# recreated at startup.  A value of "tsi1" will use a disk based index that supports higher
# cardinality datasets.
index-version = "tsi1"
```

### Minimal requirements
One F1 car stream of all samples minimum requirments:  
CPU: 8 cores  
RAM: 16Gb  
Disk: SSD min 15000 IOPS  


## InfluxDb Writer

InfluxDb writer subscribes to message brokers and saves telemetry data and session metadata in real-time to time-series and relational databases respectively. InfluxDb Writer is platform-independent and hence can be deployed on Windows or Unix-based systems as a service but **we strongly recommend deploying InfluxDb writer and InfluxDb on one Linux VM. InfluxDb performs better on Linux and by deploying it on one machine together with service you reduce unnecessary network bandwidth between those services.**

### Deployment
#### .NET Core runtime
First you need to install .NET Core 2.1 runtime. You can donwload it [here](https://www.microsoft.com/net/download/dotnet-core/2.1). Example for Ubuntu 18.04 LTE: 

```
wget -q https://packages.microsoft.com/config/ubuntu/18.04/packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb

sudo apt-get --yes install apt-transport-https
sudo apt-get update
sudo apt-get --yes install aspnetcore-runtime-2.1
```

#### Daemon installation
One the examples how to run InfluxDb writer is systemd daemon service. In the release bundle, there is a shell script **daemon_deploy.sh** for daemon installation. 

Before you run it, execute following commands:
```
awk 'BEGIN{RS="^$";ORS="";getline;gsub("\r","");print>ARGV[1]}' daemon_deploy.sh
sudo chmod 777 daemon_deploy.sh
```

Then run:
```
./daemon_deploy.sh
```

You can verify it by:

```
journalctl --unit MAT.TAP.AAS.InfluxDb.Writer.service --follow -n 100
```

or start and stop by 

```
sudo systemctl stop MAT.TAP.AAS.InfluxDb.Writer.service
sudo systemctl start MAT.TAP.AAS.InfluxDb.Writer.service
```

or **configure your config in /opt/MAT.TAP.AAS.InfluxDb.Writer/config.json**.

**Please note that after you change settings in config.json in opt folder you need to restart service.**

#### Basic usage

In order to use InfluxDb writer, add the relevant configuration in `config.json` file and start service using

    dotnet MAT.TAP.AAS.InfluxDb.Writer.dll

A sample configuration and an explanation of settings is given below.

```Json
{
  "AppConfiguration": {
    "BrokerList": "brokerList",
    "DependencyUrl": "http://localhost/api/dependencies/",
    "DependencyGroup": "dev",
    "BatchSize": 1000,
    "ThreadCount": 5,
    "MaxQueueSize": 1000,
    "InitializeDatabase": true,
    "StreamType": "Kafka", // Kafka, Mqtt
    "Username": "",
    "Password": ""
  },
  "AASConfiguration": {
    "EventTagGroup": "Tag"
  },
  "Connections": {
    "TestTopic": {
      "InfluxConnections": [
        {
          "Label": "*",
          "InfluxDbUrl": "http://localhost:8086",
          "InfluxDatabase": "Test",
          "MeasurementName": "Test"
        }
      ],
      "SqlServerConnectionString": "server=\\SQLEXPRESS;Initial Catalog=Test;User Id=UserId;Password=Password"
    }
  }
}
```
- `BrokerList`: Address of the message broker cluster.
- `DependencyUrl` and `DependencyGroup`: Settings related to ATLAS configuration and session metadata.
- `BatchSize`: Number of telemetry samples to be saved to InfluxDb at a time.
- `ThreadCount`: Number of processor threads used by the Influx Writer. A value larger than 1 can improve throughput of the writer in a machine that supports multithreading.
- `MaxQueueSize`: How many messages are processed by Influx Writer.
- `StreamType`: Type of used stream. Options are Kafka or Mqtt. 
- `Username`: Username used to establish a connection to either Kafka or Mqtt.
- `Password`: Password used to establish a connection to either Kafka or Mqtt.
- `InitializeDatabase`: True to initialize databases configured in connections section.
- `Connections`: Contains all the database connection information organized by the topic (e.g. Kafka topics.)
- `TopicName` : Name of the topic that you want to subscribe to.
- `InfluxConnections`: Contains all the InfluxDb connection strings. Influx writer supports multiple InfluxDb connections per topic under [labels](#label-supprt). If you plan to use just one InfluxDb instance, use asterisk symbol (*) as a wildcard key. This means, all telemetry data under the topic will be saved to InfluxDb specified in `InfluxDbUrl` regardless of the label.
- `SqlServerConnectionString`: Connection string for session metadata relational database. Influx Writer supports one metadata connection per topic.

#### Logging

Influx Writer has extensive logging and uses Nlog for logger implementation. You can configure logging in the `nlog.config` file or provide your own logging configuration. More information on available configuration options can be found [here](https://github.com/nlog/nlog/wiki/Configuration-file).
