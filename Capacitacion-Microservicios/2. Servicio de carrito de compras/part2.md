# Capacitación Microservicios: Módulo 2 - Logging

Para agregar logs a un proyecto de ASP.NET Core, se deben agregar los providers en la clase Startup.cs. El método CreateDefaultBuilder(args) incluye el provider para enviar logs a la consola.

Después de agregar los providers, se asigna la configuración de los logs. El método AddConfiguration agrega los ajustes cargados en la configuración del proyecto. [En la documentación de ASP.NET Core se describen los nivles de log que están disponibles.](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/logging/?view=aspnetcore-2.1#log-level) Las propiedades dentro de LogLevel determinan el nivel mínimo de log que se mostrará en la consola.

appsettings.json

```json
{
    "Logging": {
        "IncludeScopes": false,
        "LogLevel": {
            "Default": "Information",
            "System": "Information",
            "Microsoft": "Information"
        }
    }
}
```

Program.cs

```csharp

    public class Program
    {
        public static IWebHost BuildWebHost(string[] args) =>
            WebHost.CreateDefaultBuilder(args)
                .ConfigureLogging((hostingContext, builder) =>
                {
                    builder.AddConfiguration(hostingContext.Configuration.GetSection("Logging"));
                })
                .Build();
    }
```

Para escribir un log, se debe inyectar la clase ILoggerFactory dentro del constructor de la clase desde la cual se envía el mensaje y usar el método CreateLogger para obtener el logger.

```csharp

    public class RedisBasketRepository : IBasketRepository
    {
        private ILogger<RedisBasketRepository> _logger;
        private BasketSettings _settings;

        public RedisBasketRepository(IOptionsSnapshot<BasketSettings> options, ILoggerFactory loggerFactory)
        {
            _settings = options.Value;
            _logger = loggerFactory.CreateLogger<RedisBasketRepository>();
        }
    }

```

Para escribir el log, se usan los métodos del objeto _logger. [Se pueden escribir logs con cualquiera de los seis niveles disponibles.](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.logging.ilogger?view=aspnetcore-2.1)

```csharp
        private async Task ConnectToRedisAsync()
        {  
            var configuration = ConfigurationOptions.Parse(_settings.ConnectionString, true);
            configuration.ResolveDns = true;

            _logger.LogInformation($"Connecting to database {configuration.SslHost}.");
            _redis = await ConnectionMultiplexer.ConnectAsync(configuration);
        }
```

## Ejercicio

Agregue logs para registrar el proceso en el método UpdateBasketAsync de RedisBasketRepository.