# Dotnet core application with ELK stack

This is an demo using ELK stack with Filebeat and Dotnet core application

## Table of content

- [Prerequisite](#prerequisite)
- [Step 1: Create dotnet core api](#dotnetapp)
- [Step 2: Set up Filebeat](#filebeat)
- [Step 3: Install Logstash](#logstash)

## Prerequisite

- Dotnet (version 6)
- Docker
- IDE (VSCode is recommended)

## <a name="dotnetapp"></a> Step 1: Create dotnet core api 

- Create dotnet app and install logging package

```bash
# Create dotnet app
dotnet new webapi -n dotnet-elk -o app

# Navigate to /app folder
cd app

# Install required package
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.File
```

- Config logging `Program.cs` file

```dotnet
var logger = new LoggerConfiguration()
.ReadFrom.Configuration(builder.Configuration)
.Enrich.FromLogContext()
.CreateLogger();
            
builder.Logging.ClearProviders();
builder.Logging.AddSerilog(logger);
```

- Update `appsettings.json` file

```json
"Serilog": {
    "Using": [ "Serilog.Sinks.File" ],
    "MinimumLevel": {
      "Default": "Information"
    },
    "WriteTo": [
      {
        "Name": "File",
        "Args": {
          "path": "./serilogs/webapi-.log",
          "rollingInterval": "Day",
          "outputTemplate": "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} {CorrelationId} {Level:u3} {Username} {Message:lj}{Exception}{NewLine}"
        }
      },
      {
        "Name": "Console",
        "Args": {
          "outputTemplate": "{Timestamp:yyyy-MM-dd HH:mm:ss} [{Level:u3}] {Message}{NewLine}{Exception}"
        }
      }
    ]
  }
```

- Dockerize the application

## <a name="filebeat"></a> Step 2: Set up Filebeat

> Filebeat is a lightweight shipper for forwarding and centralizing log data. Installed as an agent on our servers, Filebeat monitors the log files or locations that you specify, collects log events, and forwards them either to Elasticsearch or Logstash for indexing.

- Create folder and Filebeat configuration file

```bash
mkdir filebeat
cd filebeat
```

```yml
# filebeat.yml
filebeat.inputs:
- type: filestream 
  id: dotnet-elk
  paths:
    - /usr/logs/*.log

tail_files: true

output.logstash:
  hosts: ["logstash:5044"]
```

```Docker
#Dockerfile
FROM elastic/filebeat:8.7.1
COPY --chown=root:filebeat filebeat.yml /usr/share/filebeat/filebeat.yml
```

- To verify Filebeat configuration and output run command

```bash
$ filebeat test config
Config OK

$ filebeat test output
logstash: logstash:5044...
  connection...
    parse host... OK
    dns lookup... ERROR lookup logstash on 127.0.0.11:53: read udp 127.0.0.1:48604->127.0.0.11:53: i/o timeout
```

## <a name="logstash"></a> Step 3: Install Logstash

