# Capacitación Microservicios: Módulo 3 - Comandos y querys

En un microservicio que usa CQRS, tenemos dos formas de interactuar con él.

* Enviar un comando dirigido a un aggregate del modelo de escritura
* Enviar un query dirigido al modelo de lectura

Un comando debe heredar la clase Command<TAggregate, TIdentity>, donde TAggregate es la clase del aggregate al que va dirigido, y TIdentity es la clase de identidad de ese aggregate.

El comando en sí solo es una clase que contiene la información necesaria para realizar un cambio sobre el aggregate.

```csharp
using EventFlow.Commands;

namespace Ordering.API.Application.Commands
{
    public class CancelOrderCommand : Command<Order, OrderId>
    {
        [DataMember]
        public int OrderNumber { get; private set; }

        public CancelOrderCommand(OrderId id, int orderNumber) : base(id)
        {
            OrderNumber = orderNumber;
        }
    }
}
```

El command handler define qué se va a hacer cuando se reciba el comando. El handler debe implementar la clase CommandHandler<TAggregate, TIdentity, TExecutionResult> donde TAggregate es la clase del aggregate al que va dirigido, TIdentity es la clase de identidad de ese aggregate, y TExecutionResult es la clase que representa la respuesta del comando. La clase TExecutionResult debe implementar la interfaz IExecutionResult, que contiene un valor booleano que representa si el comando se ejecutó exitosamente o no.

El método ExecuteCommandAsync se ejecuta automáticamente cuando se recibe un comando. Ese método recibe como parámetros el aggregate a modificar, los datos del commando y un token de cancelación.

El método debe retornar un execution result de la misma clase definida a la hora de heredar de CommandHandler. Este execution result se obtiene del método del aggregate con el que haremos el cambio.

Se encierra ese execution result en un Task y se retorna.

```csharp
using EventFlow.Aggregates.ExecutionResults;
using EventFlow.Commands;

namespace Ordering.API.Application.Commands
{
    public class CancelOrderCommandHandler : CommandHandler<Order, OrderId, IExecutionResult, CancelOrderCommand>
    {
        public override Task<IExecutionResult> ExecuteCommandAsync(Order aggregate, CancelOrderCommand command, CancellationToken cancellationToken)
        {
            var excecutionResult = aggregate.SetCancelledStatus();
            return Task.FromResult(excecutionResult);
        }
    }
}

```

Para enviar un comando, inyectamos la interfaz ICommandBus en la clase desde la cual lo enviaremos. Una vez hecho, se puede usar el método PublishAsync del commandBus para objeto commandBus para enviar el comando.

```csharp
namespace Services.Ordering.API.Controllers
{
    [Route("api/v1/[controller]")]
    [Authorize]
    [ApiController]
    public class OrdersController : Controller
    {
        private readonly ICommandBus _commandBus;

        public OrdersController(ICommandBus commandBus)
        {
            _commandBus = commandBus ?? throw new ArgumentNullException(nameof(commandBus));
        }

        [Route("cancel")]
        [HttpPut]
        [ProducesResponseType((int)HttpStatusCode.OK)]
        [ProducesResponseType((int)HttpStatusCode.BadRequest)]
        public async Task<IActionResult> CancelOrder([FromBody]CancelOrderCommand command, [FromHeader(Name = "x-requestid")] string requestId)
        {
            IExecutionResult result = ExecutionResult.Failed();
            if (Guid.TryParse(requestId, out Guid guid) && guid != Guid.Empty)
            {
                // Consultar número de order ....

                result = await _commandBus.PublishAsync(new CancelOrderCommand(new OrderId(order.OrderId), order.OrderNumber), CancellationToken.None);
            }

            return result.IsSuccess ? (IActionResult)Ok() : (IActionResult)BadRequest();
        }
    }
}
```

El comando también se puede enviar desde un Integration Event Handler.

```csharp
namespace Ordering.API.Application.IntegrationEvents.EventHandling
{
    public class UserCheckoutAcceptedIntegrationEventHandler : IConsumer<UserCheckoutAcceptedIntegrationEvent>
    {
        private readonly ICommandBus _commandBus;
        private readonly IPublishEndpoint _endpoint;
        private readonly ILogger<UserCheckoutAcceptedIntegrationEventHandler> _log;
        public UserCheckoutAcceptedIntegrationEventHandler(IPublishEndpoint endpoint, ICommandBus commandBus, ILogger<UserCheckoutAcceptedIntegrationEventHandler> log)
        {
            _endpoint = endpoint ?? throw new ArgumentNullException(nameof(endpoint));
            _commandBus = commandBus ?? throw new ArgumentNullException(nameof(commandBus));
            _log = log ?? throw new ArgumentNullException(nameof(log));
        }

        public async Task Consume(ConsumeContext<UserCheckoutAcceptedIntegrationEvent> context)
        {
            IExecutionResult result = ExecutionResult.Failed();

            // Generamos un nuevo Id para la orden
            var orderId = OrderId.New;

            var orderItems = from orderItem in context.Message.Basket.Items
                select new OrderStockItem(int.Parse(orderItem.ProductId), orderItem.Quantity);

            if (context.Message.RequestId != Guid.Empty)
            {
                var createOrderCommand = new CreateOrderCommand(orderId, context.Message.Basket.Items, context.Message.UserId, context.Message.UserName, context.Message.City, context.Message.Street,
                    context.Message.State, context.Message.Country, context.Message.ZipCode,
                    context.Message.CardNumber, context.Message.CardHolderName, context.Message.CardExpiration,
                    context.Message.CardSecurityNumber, context.Message.CardTypeId);

                result = await _commandBus.PublishAsync(createOrderCommand, CancellationToken.None).ConfigureAwait(false);

                // Verificamos que el comando haya sido ejecutado exitosamente
                if (result.IsSuccess)
                {
                    var orderStartedIntegrationEvent = new OrderStartedIntegrationEvent(context.Message.UserId, orderId.Value, orderItems.ToList());
                    await _endpoint.Publish(orderStartedIntegrationEvent);
                }
            }
        }
    }
}
```

Observe que el objeto OrderId tiene ciertos métodos para generar el id o sacarlo con un GUID.

Para crear un query, se crea una clase que implemente la interfaz IQuery\<TReadModel>, donde TReadModel es la clase del modelo que retorna el query.

El query debe tener como propiedades la información necesaria para hacer la consulta.

```csharp
using EventFlow.Queries;
using Ordering.ReadModel.Model;

namespace Ordering.ReadModel.Queries
{
    public class GetOrderByOrderNumberQuery : IQuery<OrderReadModel>
    {
        public GetOrderByOrderNumberQuery(int orderNumber)
        {
            this.OrderNumber = orderNumber;

        }
        public int OrderNumber { get; private set; }
    }
}
```

La consulta a realizar se define en el query handler. La clase query handler debe implementar la interfaz IQueryHandler<TQuery, TReadModel>, donde TQuery es la clase del query y TReadModel es la clase que el query retorna.

Se debe inyectar un IDbContextPRovider, que nos da el DbContext que usaremos para hacer la consulta.

El resultado de la consulta debe ser una instancia de la clase que definimos como TReadModel, y debe estar encapsulado en un Task.

```csharp
using System.Threading;
using System.Threading.Tasks;
using EventFlow.Queries;
using Ordering.ReadModel.Queries;
using EventFlow.EntityFramework;
using System.Linq;
using Ordering.ReadModel.Model;

namespace Ordering.ReadModel.QueryHandler
{
    public class GetOrderByOrderNumberQueryHandler : IQueryHandler<GetOrderByOrderNumberQuery, OrderReadModel>
    {
        private readonly IDbContextProvider<OrderingDbContext> _dbContextProvider;

        public GetOrderByOrderNumberQueryHandler(IDbContextProvider<OrderingDbContext> dbContextProvider)
        {
            _dbContextProvider = dbContextProvider;
        }

        public Task<OrderReadModel> ExecuteQueryAsync(GetOrderByOrderNumberQuery query, CancellationToken cancellationToken)
        {
            using (var context = _dbContextProvider.CreateContext())
            {
                var data = context.Orders.SingleOrDefault(x => x.OrderNumber == query.OrderNumber);
                return Task.FromResult(data);
            };
        }
    }
}
```

Para enviar el query, se debe inyectar un objeto IQueryProcessor en la clase desde la cual será enviado. Usamos el método ProcessAsync del IQueryProcessor para enviar el query.

```csharp
namespace Services.Ordering.API.Controllers
{
    [Route("api/v1/[controller]")]
    [Authorize]
    [ApiController]
    public class OrdersController : Controller
    {
        private readonly IQueryProcessor _queryProcessor;

        public OrdersController(IQueryProcessor queryProcessor)
        {
            _queryProcessor = queryProcessor ?? throw new ArgumentNullException(nameof(queryProcessor));
        }

        [Route("{orderId}")]
        [HttpGet]
        [ProducesResponseType(typeof(OrderReadModel),(int)HttpStatusCode.OK)]
        [ProducesResponseType((int)HttpStatusCode.NotFound)]
        public async Task<IActionResult> GetOrder(int orderNumber)
        {
            try
            {
                var order = await _queryProcessor.ProcessAsync(new GetOrderByOrderNumberQuery(orderId), CancellationToken.None).ConfigureAwait(false);
                return Ok(order);
            }
            catch (KeyNotFoundException)
            {
                return NotFound();
            }
        }
```

## Ejercicio

Implementar GetOrderByOrderNumberQuery.

* La clase IQuery retorna un OrderReadModel y tiene como propiedad un int OrderNumber.
* La clase IQueryHandler usa LINQ para consultar un Order con el número dado
* Enviar query en método GetOrder de OrderController

Implementar CancelOrderCommand

* La clase Command debe tener el Id de la orden como parámetro
* La clase CommandHandler debe ejecutar el método SetCancelledStatus de Order y retornar su ExecutionResult
* Enviar command en método CancelOrder de OrderController.