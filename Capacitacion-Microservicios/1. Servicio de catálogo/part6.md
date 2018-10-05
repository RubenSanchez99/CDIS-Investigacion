# Capacitación Microservicios: Servicio de catálogo

## Publicar eventos
Para reducir las dependencias entre servicios, la comunicación entre ellos se realiza mediante eventos. Estos eventos son estructuras de datos simples que contienen información sobre algo que ocurrió en el pasado (un registro se creó, una propiedad se actualizó, etcétera). Los servicios que deban realizar alguna acción después de tales sucesos pueden suscribirse al bus de eventos, el cual les notificará cuando ocurra un evento al que están suscritos.

El bus de eventos necesita un medio de transporte para enviar los eventos. En este proyecto usaremos RabbitMQ como medio de transporte.

Inicialmente, no será necesario ver la implementación del cliente de RabbitMQ, ya que usaremos la librería NServiceBus para el manejo de eventos. 

Para instalar la librería, añadimos los siguientes paquetes NuGet desde la línea de comandos.

```
dotnet add package Autofac
dotnet add package Autofac.Extensions.DependencyInjection
dotnet add package Polly
dotnet add package NServiceBus
dotnet add package NServiceBus.Persistence.Sql
dotnet add package NServiceBus.Newtonsoft.Json
dotnet add package NServiceBus.Persistence.Sql.MsBuild
dotnet add package NServiceBus.RabbitMQ
dotnet add package NServiceBus.Autofac
```

O podemos usar la extensión NuGet Package Manager para instalarlos.

```
>nuget add nservicebus
```

Cada que se agregue una librería se debe ejecutar el comando 'dotnet restore' para restaurar las nuevas dependencias. VS Code también muestra un mensaje de aviso cuando detecta un cambio en las dependencias del proyecto..

Agregue la configuración de NServiceBus en Startup.cs

#### Startup.cs
```diff
// Otros using ...

+ // Para NServiceBus
+ using Autofac;
+ using Autofac.Extensions.DependencyInjection;
+ using NServiceBus;
+ using NServiceBus.Persistence.Sql;
+ using Polly;
+ using System.Data.SqlClient;

namespace Catalog.API
{
    public class Startup
    {
-        public void ConfigureServices(IServiceCollection services)
+        public IServiceProvider ConfigureServices(IServiceCollection services)
        {
            // AddDbContext...
            // AddMvc

+           services.AddCors(options =>
+           {
+               options.AddPolicy("CorsPolicy",
+                   builder => builder.AllowAnyOrigin()
+                   .AllowAnyMethod()
+                   .AllowAnyHeader()
+                   .AllowCredentials());
+           });
+
+           var containerBuilder = new ContainerBuilder();
+
+           containerBuilder.Populate(services);
+
+           // NServiceBus
+           var container = RegisterEventBus(containerBuilder);
+
+           return new AutofacServiceProvider(container);
        }

        public void Configure(IApplicationBuilder app, IHostingEnvironment env)
        {
            // if (env.IsDevelopment)...

+           app.UseCors("CorsPolicy");
        }

+       private IContainer RegisterEventBus(ContainerBuilder containerBuilder)
+       {
+           EnsureSqlDatabaseExists();
+
+           IEndpointInstance endpoint = null;
+           containerBuilder.Register(c => endpoint)
+               .As<IEndpointInstance>()
+               .SingleInstance();
+
+           var container = containerBuilder.Build();
+
+           var endpointConfiguration = new EndpointConfiguration("Catalog");
+
+           // Configure RabbitMQ transport
+           var transport = endpointConfiguration.UseTransport<RabbitMQTransport>();
+           transport.UseConventionalRoutingTopology();
+           transport.ConnectionString(GetRabbitConnectionString());
+
+           // Configure persistence
+           var persistence = endpointConfiguration.UsePersistence<SqlPersistence>();
+           persistence.SqlDialect<SqlDialect.MsSqlServer>();
+           persistence.ConnectionBuilder(connectionBuilder:
+               () => new SqlConnection(Configuration["ConnectionString"]));
+
+           // Use JSON.NET serializer
+           endpointConfiguration.UseSerialization<NewtonsoftSerializer>();
+
+           // Enable the Outbox.
+           endpointConfiguration.EnableOutbox();
+
+           // Make sure NServiceBus creates queues in RabbitMQ, tables in SQL Server, etc.
+           // You might want to turn this off in production, so that DevOps can use scripts to create these.
+           endpointConfiguration.EnableInstallers();
+
+           // Turn on auditing.
+           endpointConfiguration.AuditProcessedMessagesTo("audit");
+
+           // Define conventions
+           var conventions = endpointConfiguration.Conventions();
+           conventions.DefiningEventsAs(c => c.Namespace != null && c.Name.EndsWith("IntegrationEvent"));
+
+           // Configure the DI container.
+           endpointConfiguration.UseContainer<AutofacBuilder>(customizations: customizations =>
+           {
+               customizations.ExistingLifetimeScope(container);
+           });
+            // Start the endpoint and register it with ASP.NET Core DI
+           endpoint = Endpoint.Start(endpointConfiguration).GetAwaiter().GetResult();
+
+           return container;
+        }
+
+       void EnsureSqlDatabaseExists()
+       {
+           var builder = new SqlConnectionStringBuilder(Configuration["ConnectionString"]);
+           var originalCatalog = builder.InitialCatalog;
+
+           builder.InitialCatalog = "master";
+           var masterConnectionString = builder.ConnectionString;
+
+           using (var connection = new SqlConnection(masterConnectionString))
+           {
+               connection.Open();
+               var command = connection.CreateCommand();
+               command.CommandText =
+                    $"IF NOT EXISTS (SELECT * FROM sys.databases WHERE name = '{originalCatalog}')" +
+                   $"  CREATE DATABASE [{originalCatalog}]";
+               command.ExecuteNonQuery();
+           }
+       }
+
+       private string GetRabbitConnectionString()
+       {
+           var host = Configuration["EventBusConnection"];
+           var user = Configuration["EventBusUserName"];
+           var password = Configuration["EventBusPassword"];
+
+           if (string.IsNullOrEmpty(user))
+               return $"host={host}";
+
+           return $"host={host};username={user};password={password};";
+       }
+
+       private Policy CreatePolicy(int retries, ILogger logger, string prefix)
+       {
+           return Policy.Handle<SqlException>().
+               WaitAndRetryAsync(
+                   retryCount: retries,
+                   sleepDurationProvider: retry => TimeSpan.FromSeconds(5),
+                   onRetry: (exception, timeSpan, retry, ctx) =>
+                   {
+                       logger.LogTrace($"[{prefix}] Exception {exception.GetType().Name} with message ${exception.Message} detected on attempt {retry} of {retries}");
+                   }
+               );
+        }
    }
}
```

También agregue el siguiente archivo de configuración a la carpeta ServiceBusBehaviors

#### ServiceBusBehaviors/OutgoingHeaderBehavior.cs
```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using NServiceBus.Features;
using NServiceBus.Pipeline;

namespace Catalog.API.ServiceBusBehaviors
{
    public class OutgoingHeaderBehavior : Behavior<IOutgoingPhysicalMessageContext>
    {
        public override Task Invoke(IOutgoingPhysicalMessageContext context, Func<Task> next)
        {
            var headers = context.Headers;

            // Remove assembly info from header based on fallback mechanism in NServiceBus
            // https://github.com/Particular/NServiceBus/blob/develop/src/NServiceBus.Core/Unicast/Messages/MessageMetadataRegistry.cs#L55
            var currentType = headers["NServiceBus.EnclosedMessageTypes"];
            var newType = currentType.Substring(0, currentType.IndexOf(','));

            headers["NServiceBus.EnclosedMessageTypes"] = newType;

            return next();
        }
    }
    public class HeaderFeature : Feature
    {
        internal HeaderFeature()
        {
            EnableByDefault();
        }

        protected override void Setup(FeatureConfigurationContext context)
        {
            var pipeline = context.Pipeline;
            pipeline.Register<OutgoingHeaderRegistration>();
        }
    }

    public class OutgoingHeaderRegistration : RegisterStep
    {
        public OutgoingHeaderRegistration() : base(
            stepId: "OutgoingHeaderManipulation",
            behavior: typeof(OutgoingHeaderBehavior),
            description: "Remove assembly info from outgoing `NServiceBus.EnclosedMessageTypes` header.")
        {
        }
    }
}
```

En la carpeta IntegrationEvents se ponen los eventos que usamos para informar a los otros servicios sobre un hecho que ha sucedido.

En la subcarpeta Events colocamos las clases que representan los eventos.



#### IntegrationEvents/Events/ProductPriceChangedIntegrationEvent.cs
```csharp
namespace Catalog.API.IntegrationEvents.Events
{
    // Integration Events notes: 
    // An Event is “something that has happened in the past”, therefore its name has to be   
    // An Integration Event is an event that can cause side effects to other microsrvices, Bounded-Contexts or external systems.
    public class ProductPriceChangedIntegrationEvent
    {        
        public int ProductId { get; private set; }

        public decimal NewPrice { get; private set; }

        public decimal OldPrice { get; private set; }

        public ProductPriceChangedIntegrationEvent(int productId, decimal newPrice, decimal oldPrice)
        {
            ProductId = productId;
            NewPrice = newPrice;
            OldPrice = oldPrice;
        }
    }
}
```

Cuando el servicio necesita realizar una acción después de detectar un evento, creamos una clase Event Handler para el evento correspondiente.

Cambia esta parte en el método UpdateProduct  CatalogController.cs para publicar un evento cuando el precio cambia.

#### Controller/CatalogController.cs
```diff
// Otros using...

+// Para el evento
+using Catalog.API.IntegrationEvents.Events;
+using NServiceBus;

namespace Catalog.API.Controllers
{
    [Route("api/v1/[controller]")]
    public class CatalogController : ControllerBase
    {
        private readonly CatalogContext _catalogContext;
+       private readonly IEndpointInstance _endpoint;

-       public CatalogController(CatalogContext context)
+        public CatalogController(CatalogContext context, IEndpointInstance endpoint)
        {
+           _endpoint = endpoint;
            _catalogContext = context ?? throw new ArgumentNullException(nameof(context));
            ((DbContext)context).ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
        }

        public async Task<IActionResult> UpdateProduct([FromBody]CatalogItem productToUpdate)
        {
            // ...

            // Update current product
            catalogItem = productToUpdate;
            _catalogContext.CatalogItems.Update(catalogItem);

            // Cambia esta linea
-           await _catalogContext.SaveChangesAsync();
+           if (raiseProductPriceChangedEvent) // Save and publish integration event if price has changed
+           {
+               //Create Integration Event to be published through the Event Bus
+               var priceChangedEvent = new ProductPriceChangedIntegrationEvent(catalogItem.Id, productToUpdate.Price, oldPrice);
+                // Publish through the Event Bus
+               await _endpoint.Publish(priceChangedEvent);
+           }
+           else // Save updated product
+           {
+               await _catalogContext.SaveChangesAsync();
+           }

            return CreatedAtAction(nameof(GetItemById), new { id = productToUpdate.Id }, null);
        }
    }
}
```

## Material extra
 * [NServiceBus Docs](https://docs.particular.net/nservicebus/)