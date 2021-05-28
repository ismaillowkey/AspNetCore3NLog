# Intruduction
ASP Net core 3.1 with Nlog and logging to CSV

## Installation
Install Nlog

```bash
Install-Package NLog -Version 4.7.10
Install-Package NLog.Web.AspNetCore -Version 4.12.0
```

## Usage
Delete Existing Nlog.config and create file nlog.config again. Set Copy to Output Directory = Copy if Newer and Replace with this code

```
<?xml version="1.0" encoding="utf-8" ?>
<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      autoReload="true"
      throwConfigExceptions="true"
      internalLogLevel="info">

	<!-- enable asp.net core layout renderers -->
	<extensions>
		<add assembly="NLog.Web.AspNetCore"/>
	</extensions>

	<!-- the targets to write to -->
	<targets>
		<!-- File Target for all log messages with basic details -->
		<target xsi:type="File" name="allfile" fileName="${basedir}/Logs/log_aspnetcore_${date:format=yyyy-MMMM-dd}.csv">
			<layout xsi:type="CsvLayout" delimiter="Comma" withHeader="true">
				<column name="Time" layout="${date:format=yyyy-MM-ddTHH\:mm\:ss.fff}" quoting="Nothing" />
				<column name="EventID" layout="${event-properties:item=EventId_Id:whenEmpty=0}" />
				<column name="Level" layout="${uppercase:${level}}" quoting="Nothing"/>
				<column name="Logger" layout="${logger}" />
				<column name="Message" layout="${message} " quoting="All" />
				<column name="Exception" layout="${exception:format=tostring}" quoting="All"/>
			</layout>
		</target>

		<!-- File Target for own log messages with extra web details using some ASP.NET core renderers -->
		<target xsi:type="File" name="ownFile-web" fileName="${basedir}/Logs/log_aspnetcoreOwin_${date:format=yyyy-MMMM-dd}.csv">
			<layout xsi:type="CsvLayout" delimiter="Comma" withHeader="true">
				<column name="Time" layout="${date:format=yyyy-MM-ddTHH\:mm\:ss.fff}" quoting="Nothing" />
				<column name="EventID" layout="${event-properties:item=EventId_Id:whenEmpty=0}" />
				<column name="Level" layout="${uppercase:${level}}" quoting="Nothing"/>
				<column name="Logger" layout="${logger}" />
				<column name="Message" layout="${message} " quoting="All" />
				<column name="Exception" layout="${exception:format=tostring}" quoting="All"/>
				<column name="URL" layout="${aspnet-request-url}"/>
				<column name="Action" layout="${aspnet-mvc-action}"/>
			</layout>
		</target>

		<!--Console Target for hosting lifetime messages to improve Docker / Visual Studio startup detection -->
		<target xsi:type="Console" name="lifetimeConsole" layout="${level:truncate=4:lowercase=true}: ${logger}[0]${newline}      ${message}${exception:format=tostring}" />
	</targets>

	<!-- rules to map from logger name tom Microsoft-->
	<logger name="*" minlevel="Trace" writeTo="allfile" />

	<!--Output host target -->
	<rules>
		<!--All logs, including from Microsoft-->
		<logger name="*" minlevel="Trace" writeTo="allfile" />

		<!--Output hosting lifetime messages to console target for faster startup detection -->
		<logger name="Microsoft.Hosting.Lifetime" minlevel="Info" writeTo="lifetimeConsole, ownFile-web" final="true" />

		<!--Skip non-critical Microsoft logs and so log only own logs-->
		<logger name="Microsoft.*" maxlevel="Info" final="true" />
		<!-- BlackHole -->

		<logger name="*" minlevel="Trace" writeTo="ownFile-web" />
	</rules>
</nlog>
```

Add code to Program.cs like this
```
public class Program
    {
        public static void Main(string[] args)
        {
            var logger = NLog.Web.NLogBuilder.ConfigureNLog("Nlog.config").GetCurrentClassLogger();
            logger.Debug("init main");
            CreateHostBuilder(args).Build().Run();
        }

        public static IHostBuilder CreateHostBuilder(string[] args) =>
            Host.CreateDefaultBuilder(args)
                .ConfigureWebHostDefaults(webBuilder =>
                {
                    webBuilder.UseStartup<Startup>();
                }).ConfigureLogging(logging =>
                {
                    logging.ClearProviders();
                    logging.SetMinimumLevel(Microsoft.Extensions.Logging.LogLevel.Trace);
                })
                .UseNLog();
    }
```

Change appsettings.json like this
```
{
  "Logging": {
    "LogLevel": {
      "Default": "Trace",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "AllowedHosts": "*"
}
```

# Example in Controller
```
private readonly ILogger<WeatherForecastController> _logger;

public WeatherForecastController(ILogger<WeatherForecastController> logger)
{
     _logger = logger;
}

[HttpGet]
public IEnumerable<WeatherForecast> Get()
{
	_logger.LogDebug("Get WeatherForecast");
	var rng = new Random();
	return Enumerable.Range(1, 5).Select(index => new WeatherForecast
	{
		Date = DateTime.Now.AddDays(index),
		TemperatureC = rng.Next(-20, 55),
		Summary = Summaries[rng.Next(Summaries.Length)]
	})
	.ToArray();
}

```

# Log Level
```
_logger.LogTrace("Log Trace");
_logger.LogDebug("Log Debug");
_logger.LogInformation("Log Info");
_logger.LogWarning("Log Warning");
_logger.LogError("Log Error");
_logger.LogCritical("Log Critical");
```
