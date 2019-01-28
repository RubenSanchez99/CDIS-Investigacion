# Capacitación Microservicios: Módulo 3 - Eventos de dominio

Antes de proseguir, lea las páginas 226-229 del libro "Microservices Architecture for Containerized .NET Applications" y la página 13 del libro ["Domain-Driven Design Reference"](http://domainlanguage.com/wp-content/uploads/2016/05/DDD_Reference_2015-03.pdf).

Los eventos de dominio realizan una notificación de un suceso ocurrido y procesado dentro del mismo microservicio. Además, forman la base de la creación de un aggregate con Event Sourcing.

Para crear un evento de dominio, se debe crear una clase que implemente la interfaz IAggregateEvent<TAggregate, TIdentity>, donde TAggregate es la clase del aggregate que publicará el evento, y TIdentity es la clase de identidad de ese aggregate.

El evento de dominio debe contener suficiente información para describir el suceso que ocurrió en el dominio. Si el evento se genera al intentar cambiar el estado de un aggregate, el evento debe tener las propiedades que cambiaron del aggregate.

```csharp
using Ordering.Domain.AggregatesModel.BuyerAggregate;
using Ordering.Domain.AggregatesModel.BuyerAggregate.Identity;
using Ordering.Domain.AggregatesModel.OrderAggregate;
using Ordering.Domain.AggregatesModel.OrderAggregate.Identity;
using EventFlow.Aggregates;

namespace Ordering.Domain.Events
{
    public class BuyerCreatedDomainEvent : IAggregateEvent<Buyer, BuyerId>
    {
        public BuyerCreatedDomainEvent(string identityGuid, string buyerName)
        {
            this.IdentityGuid = identityGuid;
            this.BuyerName = buyerName;

        }
        public string IdentityGuid { get; private set; }
        public string BuyerName { get; private set; }
    }
}
```

Para agregar EventFlow al proyecto, se debe crear un container con su configuración. Después de agregar EventFlow al container con el método EventFlowOptions.New.UseAutofacContainerBuilder(containerBuilder), se usa el método AddEvents para agregar una lista con los Type de los eventos a usar.

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
                .AddEvents(events);

            containerBuilder.Populate(services);

            return new AutofacServiceProvider(containerBuilder.Build());
        }
```

## Ejercicio

Crear evento OrderStartedDomainEvent. Recuerde agregar el evento en Startup.

* string UserId
* string UserName
* int CardTypeId
* string CardNumber
* string CardSecurityNumber
* string CardHolderName
* DateTime CardExpiration
* DateTime OrderDate
* Address Address
* IEnumerable\<OrderItem> Items