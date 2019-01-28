# Capacitación Microservicios: Módulo 3 - Aggregate con Event Sourcing

Antes de proseguir, lea la [sección de Event Sourcing de CQRS.nu](http://www.cqrs.nu/Faq/event-sourcing) y el artículo [One simple trick to make Event Sourcing click](https://blog.arkency.com/one-simple-trick-to-make-event-sourcing-click/).

Los aggregates en EventFlow son clases entidad que contienen otras clases entidades, objetos valor o enumeraciones.

Con el patrón del Event Sourcing, podemos usar los eventos de dominio para el manejo de estado de la aplicación. Cuando el modelo es adecuado para usar event sourcing, usar eventos de dominio reduce la probabilidad de no ejecer las reglas de negocio del sistema.

Para crear un aggregate con EventFlow, se crea una clase que herede AggregateRoot<TAggregate, TIdentity>, donde TAggregate es la clase del aggregate, y TIdentity es la clase de identidad de ese aggregate.

Al igual que las clases entidad, el aggregate necesita un constructor que reciba el id y ejecute el constructor base.

La alteración el estado del aggregate se divide en dos partes.

* Una llamada a un método del aggregate, que representa la intención de modificar su estado. Este método realiza validaciones, de ser necesario, y emite un evento de dominio que represente el cambio hecho.
* Un método Apply, que recibe el evento de dominio emitido y cambia el estado del aggregate.

El evento no debe ser emitido si los datos recibidos no son válidos, y no deben provocar que el aggregate quede en un estado inválido ni provocar una excepción al aplicarlos. Una vez emitidos, los eventos no podrán ser modificados.

```csharp
namespace Ordering.Domain.AggregatesModel.BuyerAggregate
{
    public class Buyer : AggregateRoot<Buyer, BuyerId>
    {
        public string IdentityGuid { get; private set; }
        public string BuyerName { get; private set; }

        public Buyer(BuyerId id) : base(id)
        {
        }

        public void Create(string identity, string name)
        {
            if (string.IsNullOrWhiteSpace(identity))
                throw new ArgumentNullException(nameof(identity));
            if (string.IsNullOrWhiteSpace(name))
                throw new ArgumentNullException(nameof(name));

            Emit(new BuyerCreatedDomainEvent(identity, name));
        }

        public void Apply(BuyerCreatedDomainEvent aggregateEvent)
        {
            IdentityGuid = aggregateEvent.IdentityGuid;
            BuyerName = aggregateEvent.BuyerName;
        }
    }
}
```

Los eventos se persiten a un event store. EventFlow tiene diferentes opciones para almacenar los eventos. En este proyecto usaremos Entity Framework Core para almacenar el evento en una base de datos de PostgreSQL.

```csharp
using Microsoft.EntityFrameworkCore;
using EventFlow.EntityFramework.Extensions;
using Microsoft.EntityFrameworkCore.Design;
using Microsoft.Extensions.Configuration;
using System;

namespace Ordering.ReadModel
{
    public class OrderingDbContext : DbContext
    {
        public OrderingDbContext(DbContextOptions<OrderingDbContext> options) : base(options)
        {
        }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            modelBuilder.AddEventFlowEvents();
        }
    }
}
```

```csharp
using System;
using Microsoft.EntityFrameworkCore;
using EventFlow.EntityFramework;
using Microsoft.Extensions.Configuration;

namespace Ordering.ReadModel
{
    public class DbContextProvider : IDbContextProvider<OrderingDbContext>, IDisposable
    {
        private readonly DbContextOptions<OrderingDbContext> _options;

        public DbContextProvider(string connectionString, IConfiguration config)
        {
            _options = new DbContextOptionsBuilder<OrderingDbContext>()
                .UseNpgsql(config["ConnectionString"])
                .Options;
        }

        public OrderingDbContext CreateContext()
        {
            var context = new OrderingDbContext(_options);
            return context;
        }

        public void Dispose()
        {
        }
    }
}
```

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
                .AddDbContextProvider<OrderingDbContext, DbContextProvider>();

            containerBuilder.Populate(services);

            return new AutofacServiceProvider(containerBuilder.Build());
        }
```

Tenga en cuenta que los eventos solo se persisirán en el event store si se emiten en una transacción con el IAggregateStore. Esto se hace automáticamente en un Command Event Handler o usando el método UpdateAsync de IAggregateStore.

## Ejercicio

Aplique los siguientes eventos al aggregate Order. Use el método StatusChangeException si se intenta cambiar el estatus de una manera inválida:

* SetStockConfirmedStatus: Emite el evento OrderStatusChangedToStockConfirmedDomainEvent si Order está en estatus AwaitingValidation.
* SetShippedStatus: Emite el evento OrderShippedDomainEvent si Order no está en estatus Paid. Retorna un executionResult.Success() si el evento se puede emitir correctamente.
* SetCancelledStatus: Emite el evento OrderCancelledDomainEvent si Order no está en estatus Shipped o Paid. Retorna un executionResult.Success() si el evento se puede emitir correctamente.

Recuerde implementar la interfaz IApply para definir el cambio de estado del aggregate al aplicar un determinado evento.