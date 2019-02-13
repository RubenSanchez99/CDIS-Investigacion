# Capacitación Microservicios: Módulo 3 - Sagas

Antes de proseguir, leer el artículo [Patterns for distributed transactions within a microservices architecture](https://developers.redhat.com/blog/2018/10/01/patterns-for-distributed-transactions-within-a-microservices-architecture/).

Las [sagas](http://masstransit-project.com/MassTransit/advanced/sagas/) son una forma de controlar una transacción que involucra múltiples servicios. La saga actúa como un autómata finito que cambia de estado en respuesta a eventos de integración.

Para crear una saga, primero se crea una clase que herede SagaStateMachineInstance. Esta clase contendrá el estado del automata finito que controla la saga.

```csharp
namespace Ordering.API.Application.Sagas
{
    public class GracePeriod : SagaStateMachineInstance
    {
        public string CurrentState { get; set; }
        public Guid CorrelationId { get ; set; }
        public string OrderId { get; set; }
        public OrderId OrderIdentity { get; set; }
        public Guid? ExpirationId { get; set; }
    }
}
```

La segunda clase que se crea hereda de MassTransitStateMachine\<T>, donde T es la clase que heredó de SagaStateMachineInstance.

```csharp
namespace Ordering.API.Application.Sagas
{
    public class GracePeriodStateMachine :
    MassTransitStateMachine<GracePeriod>
    {
        // ...
    }
}
```

Como propiedades de la clase StateMachine, se tienen los estados, eventos y schedules.

Las propiedades con la clase State representan los diferentes estados en los que puede estar el automata finito de la saga.

```csharp
        public State AwaitingValidation { get; private set; }
        public State StockConfirmed { get; private set; }
        public State PaymentSucceded { get; private set; }
        public State Validated { get; private set; }
        public State Failed { get; private set; }
```

Las propiedades de la clase Event son los eventos de integración a los que responde la saga.

```csharp
        public Event<OrderStartedIntegrationEvent> OrderStarted { get; private set; }
        public Event<OrderStockConfirmedIntegrationEvent> OrderStockConfirmed { get; private set; }
        public Event<OrderStockRejectedIntegrationEvent> OrderStockRejected { get; private set; }
        public Event<OrderPaymentSuccededIntegrationEvent> OrderPaymentSucceded { get; private set; }
        public Event<OrderPaymentFailedIntegrationEvent> OrderPaymentFailed { get; private set; }
        public Event<OrderStatusChangedToCancelledIntegrationEvent> OrderCanceled { get; private set; }
        public Event<OrderStockSentForOrderIntegrationEvent> OrderStockSent { get; private set; }
```

Las propiedades de clase Schedule\<TSagaState, TEvent> representan acciones que se realizarán después de un determinado tiempo (como por ejemplo, cancelar una orden si no se valida en 10 minutos). La clase GracePeriodExpired representa el evento que se va a arrojar con ese Schedule.

```csharp
        public Schedule<GracePeriod, GracePeriodExpired> GracePeriodFinished { get; private set; }

        public class GracePeriodExpired
        {
            public string OrderId { get; private set; }

            public GracePeriodExpired(string orderId) => OrderId = orderId;
        }
```

La lógica del autómata se pone en el constructor de la saga.

```csharp
    public class GracePeriodStateMachine :
    MassTransitStateMachine<GracePeriod>
    {
        public GracePeriodStateMachine(IAggregateStore aggregateStore)
        {
            // InstanceState inicializa el estado de la saga y asigna una propiedad del SagaStateMachineInstance para contenerlo
            InstanceState(x => x.CurrentState);

            // Event configura un evento que la saga observará.
            // Las sagas deben tener una propiedad en común con los eventos que recibe.
            // En este caso, esa propiedad de correlación es el OrderId.
            // La clase SagaStateMachineInstance tiene un atributo CorrelationId, que es un GUID que tiene el mismo propósito. El método SelectId le asigna un Guid, pero no se usará como propiedad de correlación (ya que OrderId cumple ese objetivo)
            Event(() => OrderStarted, x => x.CorrelateBy(gracePeriod => gracePeriod.OrderId, context => context.Message.OrderId).SelectId(context => Guid.NewGuid()));

            // Se configura la propiedad de correlación de los demás eventos
            Event(() => OrderStockConfirmed, x => x.CorrelateBy(gracePeriod => gracePeriod.OrderId, context => context.Message.OrderId));

            // Configuramos un Schedule que publique el evento GracePeriodFinished después de un minuto
            Schedule(() => GracePeriodFinished, x => x.ExpirationId, x =>
            {
                x.Delay = TimeSpan.FromMinutes(1);
                x.Received = e => e.CorrelateBy(gracePeriod => gracePeriod.OrderId, context => context.Message.OrderId);
            });

            // El método Initially define qué se hará en el estado inicial de la saga
            Initially(
                // El método When nos permite realizar acciones cuando ocurra un determinado evento y la saga esté en el estado definido por el método que lo rodea (en este caso, Initially)
                When(OrderStarted)
                    // El método Then define una acción síncrona que se hace cuando se recibe el evento
                    // Como es el primer evento, se inicializan los datos del estado de la saga
                    // context.Instance contiene los datos del estado de la saga (la clase SagaStateMachineInstance)
                    // context.Data contiene los datos del evento recibido
                    .Then(context => context.Instance.OrderId = context.Data.OrderId)
                    .Then(context => context.Instance.OrderIdentity = new OrderId(context.Data.OrderId))
                    // Publish publica un evento de integración
                    .Publish(context => new GracePeriodConfirmedIntegrationEvent(context.Data.OrderId))
                    // ThenAsync hace lo mismo que async, pero para métodos asíncronos
                    .ThenAsync(context =>
                        SetAwaitingValidationStatus(context.Instance.OrderIdentity, context.CreateConsumeContext().MessageId.Value.ToSourceId())
                    )
                    // Schedule inicia un temporizador que publicará un evento
                    .Schedule(GracePeriodFinished, context => new GracePeriodExpired(context.Data.OrderId))
                    // TransitionTo cambia el estado del autómata
                    .TransitionTo(AwaitingValidation)
            );

            During(StockConfirmed,
                When(OrderPaymentSucceded)
                    // Unschedule cancela un Schedule inicializado anteriormente
                    .Unschedule(GracePeriodFinished)
                    .ThenAsync(context => SetPaidStatusAsync(context.Instance.OrderIdentity, context.CreateConsumeContext().MessageId.Value.ToSourceId()))
                    .TransitionTo(Validated),
                When(OrderPaymentFailed)
                    .Unschedule(GracePeriodFinished)
                    .ThenAsync(context => SetCancelledStatusAsync(context.Instance.OrderIdentity, context.CreateConsumeContext().MessageId.Value.ToSourceId()))
                    .TransitionTo(Failed),
                When(GracePeriodFinished.Received)
                    .TransitionTo(Failed)
            );

            During(Validated,
                When(OrderStockSent)
                    .ThenAsync(context => ShipOrderAsync(context.Instance.OrderIdentity, context.CreateConsumeContext().MessageId.Value.ToSourceId()))
                    // Finalize cambia el estado del autómata a Final
                    .Finalize()
            );

            // WhenEnter define lo que se hará cuando el autómata entre a un determinado estado
            WhenEnter(Failed,
                x => x.ThenAsync(context => SetCancelledStatusAsync(context.Instance.OrderIdentity, context.CreateConsumeContext().MessageId.Value.ToSourceId()))
                      .Finalize()
            );

            During(Final,
                // Ingnore ignora el evento recibido
                Ignore(GracePeriodFinished.AnyReceived)
            );
        }
    }
```

Registramos la saga en Startup.

```csharp
        private void ConfigureMassTransit(IServiceCollection services, ContainerBuilder containerBuilder)
        {
            services.AddScoped<IHostedService, MassTransitHostedService>();
            services.AddScoped<UserCheckoutAcceptedIntegrationEventHandler>();

            containerBuilder.Register(c =>
            {
                var busControl = Bus.Factory.CreateUsingRabbitMq(sbc => 
                {
                    var host = sbc.Host(new Uri(Configuration["EventBusConnection"]), h =>
                    {
                        h.Username(Configuration["EventBusUserName"]);
                        h.Password(Configuration["EventBusPassword"]);
                    });
                    sbc.ReceiveEndpoint(host, "order_validation_state", e =>
                    {
                        e.UseRetry(x =>
                            {
                                x.Handle<DbUpdateConcurrencyException>();
                                x.Interval(5, TimeSpan.FromMilliseconds(100));
                            }); // Add the retry middleware for optimistic concurrency
                        e.StateMachineSaga(new GracePeriodStateMachine(c.Resolve<IAggregateStore>()), new InMemorySagaRepository<GracePeriod>());
                    });
                    sbc.UseInMemoryScheduler();
                });
                var consumeObserver = new ConsumeObserver(_loggerFactory.CreateLogger<ConsumeObserver>());
                busControl.ConnectConsumeObserver(consumeObserver);

                var sendObserver = new SendObserver(_loggerFactory.CreateLogger<SendObserver>());
                busControl.ConnectSendObserver(sendObserver);

                return busControl;
            })
            .As<IBusControl>()
            .As<IPublishEndpoint>()
            .SingleInstance();
        }
```

## Ejercicio

Complete la saga GracePeriod

* Registre el evento OrderStockSent (correspondiente al evento OrderStockSentForOrderIntegrationEvent) en la saga. Este evento es emitido por el servicio Catalog cuando se saca producto del inventario para una orden.
* Cree un método privado llamado ShipOrderAsync, el cual usa IAggregateRepository para cambiar el estátus de la Orden a Shipped.
* Ejecute el método ShipOrderAsync cuando la saga esté en estatus Validated y se detecte el evento OrderStockSent.

Si se realizaron los pasos correctamente en todos los ejercicios del módulo, se puede ejecutar el ciclo completo cuando se usa la petición UserCheckout en Postman. Revise la tabla EventEntity en la base de datos CapacitacionMicroservicios.OrderingDb para comprobar que se hayan generado los eventos correctamente.

El servicio WebMVC contiene una interfaz hecha con ASP.NET Core MVC. Al terminar este ejercicio, se puede probar el proyecto completo usando los servicios del módulo 3. El servicio WebMVC se habilita en la ruta "http://localhost:5100".

Si se intenta probar el servicio con una interfaz gráfica (como la interfaz Web) y el servicio Identity está levantado con Docker, es necesario configurar la variable ESHOP_EXTERNAL_DNS_NAME_OR_IP en el archivo .env para que el login funcione correctamente. Esta variable debe tener la dirección IP de la computadora donde se ejecuta el proyecto. Esto es necesario porque la interfaz no puede encontrar el archvivo discovery del servicio Identity sin la dirección externa del host.