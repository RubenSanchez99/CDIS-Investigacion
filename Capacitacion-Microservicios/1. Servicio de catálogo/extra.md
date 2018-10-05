Otros appsetting.json
```json
{
    "ConnectionStrings": {
        "DefaultConnection": "Server=XXXXXX,1433;Initial Catalog=XXXXXX;Persist Security Info=False;User ID=XXXXXXX;Password=XXXXXXX;MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;",
    }
}
```
Alternativamente
```json
{
    "ConnectionStrings": {
        "DefaultConnection": "Server=(localdb)/mssqllocaldb;Database=EFGetStarted.AspNetCore.NewDb;Trusted_Connection=True;ConnectRetryCount=0",
    }
}
```
```json
{
    "ConnectionStrings": {
        "DefaultConnection": "Server=127.0.0.1,1433;Database=Master;User Id=SA;Password=YourSTRONG!Passw0rd",
    }
}
```
El oficial
```json
{
  "ConnectionString": "Server=tcp:127.0.0.1,5433;Initial Catalog=Microsoft.eShopOnContainers.Services.CatalogDb;User Id=sa;Password=Pass@word",
  "PicBaseUrl": "http://localhost:5101/api/v1/catalog/items/[0]/pic/",
  "UseCustomizationData": false,
  "Logging": {
    "IncludeScopes": false,
    "LogLevel": {
      "Default": "Trace",
      "System": "Information",
      "Microsoft": "Information"
    }
}
```
```csharp
using EFGetStarted.AspNetCore.NewDb.Models;
using Microsoft.EntityFrameworkCore;
```
```csharp
// This method gets called by the runtime. Use this method to add services to the container.
public void ConfigureServices(IServiceCollection services)
{
    services.Configure<CookiePolicyOptions>(options =>
    {
        // This lambda determines whether user consent for non-essential cookies is needed for a given request.
        options.CheckConsentNeeded = context => true;
        options.MinimumSameSitePolicy = SameSiteMode.None;
    });

    services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_1);

    var connection = Configuration.GetConnectionString("DefaultConnection");
    services.AddDbContext<CatalogContext>(options => options.UseSqlServer(connection));
}
```

#### IntegrationEvents/Events/OrderStatusChangedToAwaitingValidationIntegrationEvent.cs
```csharp
namespace eShopOnContainers.Services.IntegrationEvents.Events
{
    using System.Collections.Generic;
    using System.Linq;

    public class OrderStatusChangedToAwaitingValidationIntegrationEvent
    {
        public int OrderId { get; set; }
        public List<OrderStockItem> OrderStockItems { get; set; }

        public OrderStatusChangedToAwaitingValidationIntegrationEvent(int orderId,
            IEnumerable<OrderStockItem> orderStockItems)
        {
            OrderId = orderId;
            OrderStockItems = orderStockItems.ToList();
        }
    }

    public class OrderStockItem
    {
        public int ProductId { get; }
        public int Units { get; }

        public OrderStockItem(int productId, int units)
        {
            ProductId = productId;
            Units = units;
        }
    }
}
```

#### IntegrationEvents/Events/OrderStatusChangedToPaidIntegrationEvent.cs
```csharp
namespace eShopOnContainers.Services.IntegrationEvents.Events
{
    using System.Collections.Generic;
    using System.Linq;

    public class OrderStatusChangedToPaidIntegrationEvent
    {
        public int OrderId { get; set; }
        public List<OrderStockItem> OrderStockItems { get; set; }

        public OrderStatusChangedToPaidIntegrationEvent(int orderId,
            IEnumerable<OrderStockItem> orderStockItems)
        {
            OrderId = orderId;
            OrderStockItems = orderStockItems.ToList();
        }
    }
}
```

#### IntegrationEvents/Events/OrderStockConfirmedIntegrationEvent.cs
```csharp
namespace eShopOnContainers.Services.IntegrationEvents.Events
{
    public class OrderStockConfirmedIntegrationEvent
    {
        public int OrderId { get; }

        public OrderStockConfirmedIntegrationEvent(int orderId) => OrderId = orderId;
    }
}
```

#### IntegrationEvents/EventHandling/OrderStatusChangedToAwaitingValidationIntegrationEventHandler.cs
```csharp
using eShopOnContainers.Services.IntegrationEvents.Events;
using NServiceBus;

namespace Microsoft.eShopOnContainers.Services.Catalog.API.IntegrationEvents.EventHandling
{
    using System.Threading.Tasks;
    using Infrastructure;
    using System.Collections.Generic;
    using System.Linq;

    public class OrderStatusChangedToAwaitingValidationIntegrationEventHandler : 
        IHandleMessages<OrderStatusChangedToAwaitingValidationIntegrationEvent>
    {
        private readonly CatalogContext _catalogContext;

        public OrderStatusChangedToAwaitingValidationIntegrationEventHandler(CatalogContext catalogContext)
        {
            _catalogContext = catalogContext;
        }

        public async Task Handle(OrderStatusChangedToAwaitingValidationIntegrationEvent message, IMessageHandlerContext context)
        {
            var confirmedOrderStockItems = new List<ConfirmedOrderStockItem>();

            foreach (var orderStockItem in message.OrderStockItems)
            {
                var catalogItem = _catalogContext.CatalogItems.Find(orderStockItem.ProductId);
                var hasStock = catalogItem.AvailableStock >= orderStockItem.Units;
                var confirmedOrderStockItem = new ConfirmedOrderStockItem(catalogItem.Id, hasStock);

                confirmedOrderStockItems.Add(confirmedOrderStockItem);
            }

            if (confirmedOrderStockItems.Any(c => !c.HasStock))
            {
                await context.Publish(new OrderStockRejectedIntegrationEvent(message.OrderId, confirmedOrderStockItems));
            }
            else
            {
                await context.Publish(new OrderStockConfirmedIntegrationEvent(message.OrderId));
            }
        }
    }
}
```

#### IntegrationEvents/Events/OrderStockRejectedIntegrationEvent.cs
```csharp
namespace eShopOnContainers.Services.IntegrationEvents.Events
{
    using System.Collections.Generic;

    public class OrderStockRejectedIntegrationEvent
    {
        public int OrderId { get; }

        public List<ConfirmedOrderStockItem> OrderStockItems { get; }

        public OrderStockRejectedIntegrationEvent(int orderId,
            List<ConfirmedOrderStockItem> orderStockItems)
        {
            OrderId = orderId;
            OrderStockItems = orderStockItems;
        }
    }

    public class ConfirmedOrderStockItem
    {
        public int ProductId { get; }
        public bool HasStock { get; }

        public ConfirmedOrderStockItem(int productId, bool hasStock)
        {
            ProductId = productId;
            HasStock = hasStock;
        }
    }
}
```

#### IntegrationEvents/EventHandling/OrderStatusChangedToPaidIntegrationEventHandler.cs
```csharp
using eShopOnContainers.Services.IntegrationEvents.Events;
using NServiceBus;

namespace Microsoft.eShopOnContainers.Services.Catalog.API.IntegrationEvents.EventHandling
{
    using System.Threading.Tasks;
    using Infrastructure;

    public class OrderStatusChangedToPaidIntegrationEventHandler : 
        IHandleMessages<OrderStatusChangedToPaidIntegrationEvent>
    {
        private readonly CatalogContext _catalogContext;

        public OrderStatusChangedToPaidIntegrationEventHandler(CatalogContext catalogContext)
        {
            _catalogContext = catalogContext;
        }

        public async Task Handle(OrderStatusChangedToPaidIntegrationEvent message, IMessageHandlerContext context)
        {
            //we're not blocking stock/inventory
            foreach (var orderStockItem in message.OrderStockItems)
            {
                var catalogItem = _catalogContext.CatalogItems.Find(orderStockItem.ProductId);

                catalogItem.RemoveStock(orderStockItem.Units);
            }

            await _catalogContext.SaveChangesAsync();
        }
    }
}
```

Dockerfile
```dockerfile
FROM microsoft/dotnet:2.1-sdk AS build
COPY . ./CRUD
WORKDIR /CRUD/
RUN dotnet build -c Release -o output

FROM microsoft/dotnet:2.1-runtime AS runtime
COPY --from=build /CRUD/output .
ENTRYPOINT ["dotnet", "CRUD.dll"]
```
```dockerfile
FROM microsoft/dotnet:sdk AS build-env
WORKDIR /app

# Copy csproj and restore as distinct layers
COPY *.csproj ./
RUN dotnet restore

# Copy everything else and build
COPY . ./
RUN dotnet publish -c Release -o out

# Build runtime image
FROM microsoft/dotnet:aspnetcore-runtime
WORKDIR /app
COPY --from=build-env /app/out .
ENTRYPOINT ["dotnet", "aspnetapp.dll"]
```

```
docker run -e 'ACCEPT_EULA=Y' -e 'SA_PASSWORD=Pass@word' -p 5433:1433 -d microsoft/mssql-server-linux:2017-CU8
```