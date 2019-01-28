# Capacitación Microservicios: Módulo 3 - Modelo de lectura

La forma más sencilla de realizar consultas a un microservicio que usa Event Sourcing es generar un modelo de lectura separado que se acualize cuando detecte la publicación de un evento de dominio.

Este modelo de lectura se puede hacer con una base de datos relacional, lo que nos permite realizar consultas mediante SQL o LINQ, mismas que no podríamos realizar hacia el modelo de escritura (ya que no tiene el concepto de tablas o columnas, solo una lista de eventos).

EventFlow nos permite usar Entity Framework Core para construir el modelo de lectura.

El primer paso es crear una clase que represente nuestro modelo de lectura. Esta clase puede ser diferente al modelo que definimos en el aggregate. No está ligada al modelo de escritua hecho anteriormente, por lo que tenemos la flexibilidad de construirla de la manera que sea más útil a la hora de hacer querys.

La clase debe implementar la interfaz IReadModel. También debe implementar la interfaz IAmReadModelFor<TAggregate, TIdentity, TDomainEvent>, donde TAggregate es la clase del aggregate del modelo de escritura, TIdentity es la clase de identidad de ese aggregate y TDomainEvent es el evento de dominio que provocará un cambio en el modelo de lectura.

La clase que implementa IReadModel es una clase de entidad de Entity Framework Core, por lo que podemos usar las mismas propiedades y configuraciones que hacemos al usar EF Core.

Al implementar la interfaz IAmReadModelFor, se agregará un método Apply. Al igual que en el modelo de escritura, en este metodo se define qué propiedades de la clase cambian al ocurrir el evento.

```csharp
using EventFlow.ReadStores;

namespace Ordering.ReadModel.Model
{
    public class OrderReadModel : IReadModel,
    IAmReadModelFor<Order, OrderId, OrderStatusChangedToAwaitingValidationDomainEvent>
    {
        public string Status { get; set; }

        public void Apply(IReadModelContext context, IDomainEvent<Order, OrderId, OrderStatusChangedToAwaitingValidationDomainEvent> domainEvent)
        {
            this.Status = OrderStatus.AwaitingValidation.Name;
        }
    }
```

Agreagmos el DbSet correspondiente a la clase del modelo de lectura al DbContext.

```csharp
namespace Ordering.ReadModel
{
    public class OrderingDbContext : DbContext
    {
        public OrderingDbContext(DbContextOptions<OrderingDbContext> options) : base(options)
        {
        }

        public DbSet<OrderReadModel> Orders { get; set; }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            modelBuilder.AddEventFlowEvents();
        }
    }
}
```

Creamos una clase con un método de extensión para registrar el modelo de lectura.

```csharp
namespace Ordering.ReadModel
{
    public static class ReadModelConfiguration
    {
        public static IEventFlowOptions AddEntityFrameworkReadModel(this IEventFlowOptions efo)
        {
            return efo
                .UseEntityFrameworkReadModel<OrderReadModel, OrderingDbContext>();
        }
    }
}
```

Usamos el método de extenión en la configuración de EventFlow para registrar el modelo de lectura.

```csharp
        public IServiceProvider ConfigureServices(IServiceCollection services)
        {
            var containerBuilder = new ContainerBuilder();

            var events = new List<Type>() {
                typeof(BuyerCreatedDomainEvent)
            };

            var container = EventFlowOptions.New
                .UseAutofacContainerBuilder(containerBuilder)
                .UseConsoleLog()
                .AddEvents(events)
                .UseEntityFrameworkEventStore<OrderingDbContext>()
                .ConfigureEntityFramework(EntityFrameworkConfiguration.New)
                .AddEntityFrameworkReadModel()
                .AddDbContextProvider<OrderingDbContext, DbContextProvider>();

            containerBuilder.Populate(services);

            return new AutofacServiceProvider(containerBuilder.Build());
        }
```

## Ejercicio

Implementar query de 