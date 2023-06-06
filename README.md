# code

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