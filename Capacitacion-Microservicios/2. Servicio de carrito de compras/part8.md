# Capacitación Microservicios: Módulo 2 - Eventos de integración

Cuando un microservicio necesita comunicar a otro un suceso ocurrido durante el proceso, lo hace mediante un evento de integración.

Los eventos de integración se envían a través de un event bus; un canal compartido al que todos los microservicios tienen acceso. El mensaje se serializa como JSON y se envía al bus. Los microservicios pueden suscribirse a este evento para recibir una notificación cuando se haya publicado en el bus.

Para el manejo de eventos de integración usaremos la librería [MassTransit](http://masstransit-project.com/MassTransit/).

La librería necesita un medio de transporte en el cual se muevan los mensajes. En nuestro proyecto usaremos [RabbitMQ](https://www.rabbitmq.com/) para este fin.

Para usar MassTransit, necesitamos crear un hosted service que inicie y deshabilite el bus. Esta función la cumple el MassTransitHostedService.

```csharp
    public class MassTransitHostedService : Microsoft.Extensions.Hosting.IHostedService
    {
        private readonly IBusControl busControl;

        public MassTransitHostedService(IBusControl busControl)
        {
            this.busControl = busControl;
        }

        public async Task StartAsync(CancellationToken cancellationToken)
        {
            //start the bus
            await busControl.StartAsync(cancellationToken);
        }

        public async Task StopAsync(CancellationToken cancellationToken)
        {
            //stop the bus
            await busControl.StopAsync(TimeSpan.FromSeconds(10));
        }
    }
```

Agregamos el servicio en ConfigureServices.

```csharp
services.AddScoped<IHostedService, MassTransitHostedService>();
```

Para poder generar logs con la información de los eventos de integración, agregamos dos clases observer en la carpeta Services.

Un IConsumeObserver se ejecuta cuando el microservicio detecta un evento para consumir. Al implementar IConsumerObserver, se generarán métodos para definir la lógica a realizar antes, después y en caso de fallar el consumo del evento.

Podemos acceder a información del evento, así como su remitente, mediante la clase ConsumeContext.

```csharp
    public class ConsumeObserver: IConsumeObserver
    {
        private readonly ILogger<ConsumeObserver> _log;

        public ConsumeObserver(ILogger<ConsumeObserver> log)
        {
            _log = log;
        }

        Task IConsumeObserver.PreConsume<T>(ConsumeContext<T> context)
        {
            // called before the consumer's Consume method is called
            _log.LogInformation($"{context.Message.GetType().Name}: {JsonConvert.SerializeObject(context.Message, Formatting.Indented)} - Attempt no. {context.GetRetryAttempt()}");
            return Task.CompletedTask;
        }
    }
```

El ISendObserver sirve para agregar lógica cuando el microservicio publica un evento.

```csharp
    public class SendObserver : ISendObserver
    {
        private readonly ILogger<SendObserver> _log;

        public SendObserver(ILogger<SendObserver> log)
        {
            _log = log;
        }

        public Task PostSend<T>(SendContext<T> context) where T : class
        {
            _log.LogInformation($"{context.Message.GetType().Name}: {JsonConvert.SerializeObject(context.Message, Formatting.Indented)}");
            return Task.CompletedTask;
        }
    }
```

Para registras MassTransit, se crea un ContainerBuilder mediante la librería [Autofac](https://autofac.org/). Dentro de este Container se registra el servicio de MassTransit y los registrados en el IServiceCollection.

Recuerde que _loggerFactory es un ILoggerFactory que se inyecta en el constructor de la clase Startup.

```csharp
        public IServiceProvider ConfigureServices(IServiceCollection services)
        {
            // Otros servicios registrados en services

            var builder = new ContainerBuilder();

            builder.Register(c =>
            {
                var busControl = Bus.Factory.CreateUsingRabbitMq(sbc =>
                    {
                        var host = sbc.Host(new Uri(Configuration["EventBusConnection"]), h =>
                        {
                            h.Username(Configuration["EventBusUserName"]);
                            h.Password(Configuration["EventBusUserName"]);
                        });
                        sbc.UseExtensionsLogging(_loggerFactory);
                    }
                );
                var consumeObserver = new ConsumeObserver(_loggerFactory.CreateLogger<ConsumeObserver>());
                busControl.ConnectConsumeObserver(consumeObserver);

                var sendObserver = new SendObserver(_loggerFactory.CreateLogger<SendObserver>());
                busControl.ConnectSendObserver(sendObserver);

                return busControl;
            })
            .As<IBusControl>()
            .As<IPublishEndpoint>()
            .SingleInstance();
            builder.Populate(services);
            var container = builder.Build();

            // Create the IServiceProvider based on the container.
            return new AutofacServiceProvider(container);
        }
```

Para publicar un evento de integración, se debe crear una clase POCO que contenga su información correspondiente. Trate de que los eventos tengan la mínima información necesaria que necesiten comunicar a otros microservicios.

Para que dos clases de eventos de integración sean detectadas como el mismo evento, deben tener el mismo nombre y namespace. No necesitan tener exactamente las mismas propiedades, pero obviamente deben tener propiedades en común para tener uso.

```csharp
namespace eShopOnContainers.Services.IntegrationEvents.Events
{
    public class ProductPriceChangedIntegrationEvent
    {
        public ProductPriceChangedIntegrationEvent(int productId, decimal newPrice, decimal oldPrice)
        {
            this.ProductId = productId;
            this.NewPrice = newPrice;
            this.OldPrice = oldPrice;
        }

        public int ProductId { get; }

        public decimal NewPrice { get; }

        public decimal OldPrice { get; }
    }
```

Los eventos se publican mediante un IPublishEndpoint, que se inyecta en el constructor de la clase desde la cual se envía el evento.

```csharp
    [Route("api/v1/[controller]")]
    public class CatalogController : ControllerBase
    {
        private readonly CatalogContext _catalogContext;
        private readonly IPublishEndpoint _endpoint;
        private readonly CatalogSettings _settings;

        public CatalogController(CatalogContext context, IPublishEndpoint endpoint, IOptionsSnapshot<CatalogSettings> settings)
        {
            _endpoint = endpoint;
            _catalogContext = context ?? throw new ArgumentNullException(nameof(context));
            _settings = settings.Value;
            ((DbContext)context).ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
        }

        //PUT api/v1/[controller]/items
        [Route("items")]
        [HttpPut]
        public async Task<IActionResult> UpdateProduct([FromBody]CatalogItem productToUpdate)
        {
            // Lógica del ActionMethod...

            var productPriceChangedIntegrationEvent = new ProductPriceChangedIntegrationEvent
            {
                ProductId = catalogItem.Id,
                NewPrice = productToUpdate.Price,
                OldPrice = oldPrice
            };

            await _endpoint.Publish<ProductPriceChangedIntegrationEvent>(productPriceChangedIntegrationEvent);
        }
    }
```

Para que un microservicio se suscriba al evento de integración, debe tener dentro de su proyecto una clase con el mismo nombre y namespace.

La lógica de qué se hace en respuesta al evento detectado se pone en un Integration Event Handler. Para crear un event handler, se implementa la interfaz IConsumer\<T>, donde T es el evento de integración que se va a consumir.

```csharp
    public class ProductPriceChangedIntegrationEventHandler : IConsumer<ProductPriceChangedIntegrationEvent>
    {
        public async Task Consume(ConsumeContext<ProductPriceChangedIntegrationEvent> context)
        {
            // Accedemos al contendido del mensaje con context.Message
        }
    }
```

Por último, se registra la suscripción en la clase Startup.

```csharp
            builder.Register(c =>
            {
                var busControl = Bus.Factory.CreateUsingRabbitMq(sbc => 
                {
                    var host = sbc.Host(new Uri(Configuration["EventBusConnection"]), h =>
                    {
                        h.Username(Configuration["EventBusUserName"]);
                        h.Password(Configuration["EventBusUserName"]);
                    });
                    // Registramos la suscripción y le asignamos un nombre a la cola de mensajes donde se guardan los eventos recibidos
                    // Consumer define en event handler a usar en para esa suscripción
                    sbc.ReceiveEndpoint(host, "price_updated_queue", e =>
                    {
                        e.Consumer<ProductPriceChangedIntegrationEventHandler>(c);
                    });
                    sbc.UseExtensionsLogging(_loggerFactory);
                });
                var consumeObserver = new ConsumeObserver(_loggerFactory.CreateLogger<ConsumeObserver>());
                busControl.ConnectConsumeObserver(consumeObserver);

                var sendObserver = new SendObserver(_loggerFactory.CreateLogger<SendObserver>());
                busControl.ConnectSendObserver(sendObserver);

                return busControl;
            })
```

## Ejercicio

Agregue MassTransit al servicio Basket.

En el servicio Catalog, publique el evento ProductPriceChangedIntegrationEvent si al actualizar un producto, su precio cambia.

En el servicio Basket, actualice todos los baskets si se detecta el evento ProductPriceChangedIntegrationEvent.

## Extra

* Libro Microsoft, página 134