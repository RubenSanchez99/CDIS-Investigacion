# Capacitación Microservicios: Módulo 3 - Suscribers

Podemos agregar código que se ejecute fuera de los aggregates en respuesta a un evento de dominio mediante subscribers.

Para agregar un subscriber, se debe crear una clase que implemente la interfaz ISubscribeSynchronousTo<TAggregate, TIdentity, TDomainEvent>, donde TAggregate es la clase del aggregate del modelo de escritura, TIdentity es la clase de identidad de ese aggregate y TDomainEvent es el evento de dominio al que se va a responder.

Para acceder al aggregate fuera de un command handler, hacemos uso del IAgggregateStore. Este objeto actúa como nuestro repositorio de aggregates. Para cargar un aggregate, usamos el método LoadAsync, enviando como parámetro el objeto Id que corresponde al aggregate deseado.

```csharp
namespace Ordering.API.Application.Subscribers.OrderPaid
{
    public class OrderStatusChangedToPaidSubscriber
        : ISubscribeSynchronousTo<Order, OrderId, OrderStatusChangedToPaidDomainEvent>
    {
        private readonly IAggregateStore _aggregateStore;
        private readonly IPublishEndpoint _endpoint;
        public OrderStatusChangedToPaidSubscriber(IAggregateStore aggregateStore, IPublishEndpoint endpoint)
        {
            _aggregateStore = aggregateStore;
            _endpoint = endpoint;
        }
        public async Task HandleAsync(IDomainEvent<Order, OrderId, OrderStatusChangedToPaidDomainEvent> domainEvent, CancellationToken cancellationToken)
        {
            var order = await _aggregateStore
                .LoadAsync<Order, OrderId>(domainEvent.AggregateIdentity, CancellationToken.None)
                .ConfigureAwait(false);

            var buyer =  await _aggregateStore
                .LoadAsync<Buyer, BuyerId>(order.GetBuyerId, CancellationToken.None)
                .ConfigureAwait(false);

            var orderStockList = domainEvent.AggregateEvent.OrderItems
                .Select(orderItem => new OrderStockItem(orderItem.ProductId, orderItem.Units));

            var orderStatusChangedToPaidIntegrationEvent = new OrderStatusChangedToPaidIntegrationEvent(
                domainEvent.AggregateIdentity.Value,
                order.OrderStatus.Name,
                buyer.BuyerName,
                orderStockList);

            await _endpoint.Publish(orderStatusChangedToPaidIntegrationEvent);  
        }
    }
}
```

Para actualizar un aggregate fuera de un command handler, usamos el método UpdateAsync del IAggregateStore. Ese método recibe como parámetros el id del aggregate, un id para diferenciar la transacción y una función donde se ejecutan los métodos del aggregate para realizar el cambio.

```csharp
namespace Ordering.API.Application.IntegrationEvents.EventHandling
{
    public class OrderPaymentFailedIntegrationEventHandler : IConsumer<OrderPaymentFailedIntegrationEvent>
    {
        private readonly IAggregateStore _aggregateStore;

        public OrderPaymentFailedIntegrationEventHandler(IAggregateStore aggregateStore)
        {
            _aggregateStore = aggregateStore ?? throw new ArgumentNullException(nameof(aggregateStore));
        }

        public async Task Consume(ConsumeContext<OrderPaymentFailedIntegrationEvent> context)
        {
            var orderId = new OrderId(context.Message.OrderId);

            await _aggregateStore.UpdateAsync<Order, OrderId>(orderId, SourceId.New,
                (order, c) => {
                        order.SetCancelledStatus();
                        return Task.FromResult(0);
                }, CancellationToken.None
            ).ConfigureAwait(false);
        }
    }
}

```

## Ejercicio

Agregar OrderStatusChangedToStockConfirmedSubscriber. El método consulta el aggregate Order y su correspondiente Buyer mediante IAggregateRepository. Publica el evento de integración OrderStatusChangedToStockConfirmedIntegrationEvent.

Los suscribers de todo el proyecto se pueden agregar mediante el assembly, usando el siguiente método en Startup.

```csharp
            var container = EventFlowOptions.New
                .UseAutofacContainerBuilder(containerBuilder)
                .UseConsoleLog()
                .AddEvents(events)
                .AddCommandHandlers(commandHandlers)
                // Agregamos todos los suscribers que estén en el mismo proyecto que la clase Startup
                .AddSubscribers(typeof(Startup).Assembly)
```