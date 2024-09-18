# sdLitica

## Preconditions
Need to install:
* MariaDB or MySQL
* InFluxDB 2.x
* RabbitMQ (+erlang)
* .NET 6 SDK

## How to build
Run command from root directory

    .\core\Apps\WebAPI\sdLitica.WebAPI\dotnet build

## How to run
Before the first start of the application it's necessary to configure `sdliticadb` database in MySQL. For this run scripts in [Database](https://github.com/sdLiticaProject/code/tree/master/core/Database) directory.

To configure InfluxDB set the credentials (InfluxToken and InfluxOrgId) in file [appsettings.Development.json](https://github.com/sdLiticaProject/code/blob/compatibility-with-influxdb-v2.0/core/Apps/WebAPI/sdLitica.WebAPI/appsettings.Development.json).

To run app execute command

    .\core\Apps\WebAPI\sdLitica.WebAPI\dotnet run

## Preprod Version
Link: https://sdlitica-preprod.cloud.cosm-lab.science/

# Overview

This section contains overall architecture of sdLitica Time-series analysis platform.

# General overview
![Solution High-level overview](https://github.com/sdLiticaProject/docs/raw/master/schemas/export/sdLitica-HighLevelOverview.png)

## Components definition
### REST API handlers
**REST API Management Front-end** - Handlers and services which provide ability to perform management operations like user registration, profile management, authorization (obtaining token) and so on.

**REST API Analytics Front-end** - Handlers and services which provide ability to manage analytic works for current user. This API will allow to see the history of competed requests with respected results and track progress of ongoing works. API provides an ability to start computation of selected kinds of analytical computations for the uploaded data.

**REST API Data-Push Front-end** - Handlers and services allowing end-user to upload (in various formats like CSV or JSON) or stream time-series data to the system.

### Core services
**Platform Core** - Key management services used to process management data in the system, like users information, metadata of uploaded user-data (time-series), metadata for analytics results and so on

**Analytiics Management Core** - Set of platfrom services for processing analytics requests and retrieving analytics results for the logged in user

**Data-Push Core** - Services for transfromation data from user formats to common system format of the time-series structures to be stored in centralized time-series storage

### Key storage components
**SQL database** - is used for general management infromation and metadata about uploaded users data (time-series) and nalytics requests processings

**NoSQL Storage** - is used to store free-from results of analytics execution. Since different kinds of execution may result in different kinds of results - free-form NoSQL storage provides more flexibility to persist them

**TimeSeries Database** - is very efficient storage for time-series data. In this storage such kind of data can be stored in a very efficient way (from space consumption perspective) and can be well searched and retrieved quickly.

### Data Processing Network (DPN)
**Data Processing Network (DPN)** - is a distributed system focused on distributed processing of analytics requests on end-user data. This is a Jenkins-like group of executors which are managed and synchronized via efficient messages-based RabbitMQ cluster.

## Solution Architecture based on Components Definition
The following image reflects layers, components and projects of the solution code. Moreover, the dependecy of the layers are illustrated by the arrows. 
In the code, the solution itselfs corresponds a `.sln` file, the layers correspond to folders and, components and projects correspond to `.csproj`.

![Solution Architecture](https://github.com/sdLiticaProject/docs/raw/master/schemas/export/sdLitica-SolutionArchitecture.png)


### Layers
The layers of the architecture were created with the aim of decouple each component and separate them based on responsabilities:

**Apps** - This layer should receives only projects that are runnable, i.e., where the Target Framework is `netcoreapp2.2` or similar, and the project corresponds to a REST interface or other project that can be published. These project should reference each corresponding `bootstrap` project. For instance, `sdLitica.Management.WebAPI` will only reference `sdLitica.Management.Bootstrap` project. 

**Infrastruture** - This layer should receives cross-cutting projects, i.e., those projects that aggregate common solution classes, used by many others projects, such as `Utils`, `Exceptions`, `Logs`, `Helpers` and others. 
Here an exception exists and it is the `bootstrap` project: this project should *never* be referenced by any other project than corresponding project from `Apps` layer. And, the `bootstrap` project should reference every project that must be bootstraped by itself (e.g. services, repositories, etc). Mainly, the `bootstrap` project configures services for dependency injection.

**Services** - This layer contains main core projects. These projects should comprise any classes that run business procedure or any amount of code that uses Entities, Repositories, general data or heavy processing. Further, the `controller` classes from `Apps` layer (inside REST projects) should reference directly a `service` class, or through an `interface`. Do not put great amount of code into `controllers`, put into `services`.

**Domain** - This layer contains projects with definitions from Entities, interfaces from repositories (data contracts) and any other classes that represent a `Model` class. These projects should *not* reference any other project than `infrastructure` projects. Also, the domain classes should be cohesive.

**Data** - This layer should contains projects that access data sources directly, i.e., SQL database, NoSQL documents, TimeSeries database. Please, note that only these projects should reference external libraries such as MySQL or MongoDB or InfluxDB libraries. Normally, a class into data project should implement an interface from the domain layers, with its data manipulation behavior/contract (methods).

## Information model
This section contains definition of major work flows and details of key data flows in sdLitica platform.

At a glance, information flows can be depicted with following schema:
![Solution High-level overview](https://github.com/sdLiticaProject/docs/raw/master/schemas/export/sdLitica-InformationModel.png )

There are two kinds on onformation flows that can be accepted by sdLitica platform:
1. Time-series snapshots - when data uploaded to the cloud as a fixed time-series dump with start and end. This can be a JSON or CSV file of somesing else.
2. Stream of time-seried data. For exaple, this can be a results of long running model execution or monitoring of some IoT-enabled system.

When time-series arrives to the cloud, first of all they are passed to data mapper. Data mapper allows to turn some free-form incoming data into standartized time-series according to pre-defined rules for this particular snapshot or stream.

After normalization, time-series data is stored in specializad database. There is a variety of databases from which we can pick one.

In case of time-series snapshots, when time-series uploaded to the service various analytics mechanisms can be applied to it.

In case of streamed data, which are also stored in time-series storage as an extension to the analysis of what is present in storage, end-user can configure number of anaytic processorts which will do analysis on streamed data on the fly.

# Static data upload

This is the simplest way to push time series data into our platfrom. Static means that end-user uploads a data snapshot into our system in one of supported fromats. We can support following:
 - CSV file with headers
   - uploaded as Multi-part request
   - uploaded as a request payload
 - JSON time-series 
   - Slim JSON-timeseries (array of two arrays) - [StackOverflow](https://stackoverflow.com/a/30243605)
     - uploaded as Multi-part request
     - uploaded as a request payload
   - JTS JSON-timeseries - [JTS from eagle.io](https://docs.eagle.io/en/latest/reference/historic/jts.html#:~:text=JSON%20Time%20Series-,JSON%20Time%20Series,exported%20via%20the%20HTTP%20API.)
     - uploaded as Multi-part request
     - uploaded as a request payload

# Incoming data streaming

Next important way of data retrieval is supporting an incoming data stream. In this scenario, user will create a time-series and set its type to incoming-stream. After that we will create a unique URL that will be albe to accept incoming data stream via sequence of POST requests or one single long staying POST request

# Data Snooping

In this case, our platform will poll a remote accessible data source with configured cadence. When user will create new time series he will set its type to snooping. For this kind of time seriece, following configurations will be required:
  - Remote URL to be polled
  - Credentials (if needed) //TBD credentials types to support
  - Data transformation function. Following options will be supported:
    - Remote XML to JTS (First we will transfrom XML -> JSON and then apply json2json)
    - Remote JSON to JTS (via json2json)

To do json2json transfromation we will use [JOLT](https://github.com/bazaarvoice/jolt), [JSLT](https://github.com/schibsted/jslt) or something else
