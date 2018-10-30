# Eventos de dominio y eventos de integración

## Eventos de integración
El propósito de este documento es explicar la forma en la que se realiza la comunicación entre microservicios mediante eventos de integración, observar sus diferencias con los eventos de dominio y dar mi punto de vista sobre cómo reducir la complejidad de los modelos de los microservicios.

En el libro '.NET Microservices Architecture for Containerized .NET Applications' se describe el concepto de comunicación basada en eventos de la siguiente forma:

```
As described earlier, when you use event-based communication, a microservice publishes an event when something notable happens, such as when it updates a business entity. Other microservices subscribe to those events. When a microservice receives an event, it can update its own business entities, which might lead to more events being published. This is the essence of the eventual consistency concept. This publish/subscribe system is usually performed by using an implementation of an event bus. The event bus can be designed as an interface with the API needed to subscribe and unsubscribe to events and to publish events. It can also have one or more implementations based on any inter-process or messaging communication, such as a messaging queue or a service bus that supports asynchronous communication and a publish/subscribe model.
```

El mismo libro da la definición de eventos de integración.

```
Integration events are used for bringing domain state in sync across multiple microservices or external systems. This is done by publishing integration events outside the microservice. When an event is published to multiple receiver microservices (to as many microservices as are subscribed to the integration event), the appropriate event handler in each receiver microservice handles the event.
```

De este modo, cuando un microservicio necesita informar a otros sobre un hecho ocurrido en su funcionamiento, se publica un evento al event bus. Otros microservicios pueden suscribirse a ese evento. Cuando el suscritor detecta que ese evento ocurrió, se manda a llamar una clase EventHandler, que realiza el flujo que ocurre en respuesta al evento. 

En la clase EventHandler también se pueden emitir nuevos eventos de integración (aunque es recomendable que no se emitan múltiples eventos que requieran estar en cierto orden, ya que no se puede garantizar que los suscritores van a recibir esos eventos en el mismo orden en el que se emitieron).

Un ejemplo de comunicación basada en eventos es el cambio de precio en el servicio de catálogo de eShopOnContainers. Cuando se actualiza un producto, se revisa si el precio fue uno de los atributos cambiados. De ser así, se emite un evento ProductPriceChanged.

El servicio de carrito de compras (basket) está suscrito a ese evento. Cuando se recibe el evento, la clase EventHandler cambia el precio del producto modificado en todos los carritos de compra.

De este modo, la comunicación entre servicios se mantiene desenlazada e independiente.

## Eventos de dominio
El libro ofrece la siguiente definición de eventos de dominio.

```
An event is something that has happened in the past. A domain event is, logically, something that happened in a particular domain, and something you want other parts of the same domain (in- process) to be aware of and potentially react to.

An important benefit of domain events is that side effects after something happened in a domain can be expressed explicitly instead of implicitly. Those side effects must be consistent so either all the operations related to the business task happen, or none of them. In addition, domain events enable a better separation of concerns among classes within the same domain.

For example, if you are just using Entity Framework and entities or even aggregates, if there have to be side effects provoked by a use case, those will be implemented as an implicit concept in the coupled code after something happened (that is, you must see the code to know there are side effects). But, if you just see that code, you might not know if that code (the side effect) is part of the main operation or if it really is a side effect. On the other hand, using domain events makes the concept explicit and part of the ubiquitous language. For example, in the eShopOnContainers application, creating an order is not just about the order; it updates or creates a buyer aggregate based on the original user, because the user is not a buyer until there is an order in place. If you use domain events, you can explicitly express that domain rule based in the ubiquitous language provided by the domain experts.

Domain events are somewhat similar to messaging-style events, with one important difference. With real messaging, message queuing, message brokers, or a service bus using AMPQ, a message is always sent asynchronously and communicated across processes and machines. This is useful for integrating multiple Bounded Contexts, microservices, or even different applications. However, with domain events, you want to raise an event from the domain operation you are currently running, but you want any side effects to occur within the same domain.

The domain events and their side effects (the actions triggered afterwards that are managed by event handlers) should occur almost immediately, usually in-process, and within the same domain. Thus, domain events could be synchronous or asynchronous. Integration events, however, should always be asynchronous.
```

La diferencia clave es que los eventos de dominio se refieren a eventos que ocurren dentro de un mismo dominio (que suele ser dentro de un mismo microservicio) y los eventos de integración se refieren a eventos que ocurrieron en otros contextos (que se originan de otros microservicios).

La aplicación de ejemplo eShopOnContainers maneja los siguientes eventos de integración.

Catalog Integration Events
  * Subscribes to:
    - OrderStatusChangedToAwaitingValidation
    - OrderStatusChangedToPaid
  * Publishes:
    - ProductPriceChanged
    - OrderStockRejected
    - OrderStockConfirmed

Basket Integration Events
  * Subscribes to:
    - OrderStarted
    - ProductPriceChanged
  * Publishes:
    - UserCheckoutAccepted
  
Order Integration Events
  * Subscribes to:
    - GracePeriodConfirmed
    - OrderPaymentFailed
    - OrderPaymentSucceded
    - OrderStockConfirmed
    - OrderStockRejected
    - UserCheckoutAccepted
  * Publishes:
    - OrderStatusChangedToCancelled
    - OrderStatusChangedToAwaitingValidation
    - OrderStatusChangedToPaid
    - OrderStatusChangedToShipped
    - OrderStatusChangedToSubmitted
    - OrderStatusChangedToStockConfirmed

Payment Integration Events
  * Subscribes to:
    - OrderStatusChangedToStockConfirmed
  * Publishes:
    - OrderPaymentSucceded
    - OrderPaymentFailed

El servicio Orders es el único que maneja un modelo de dominio con CQRS, por lo que es el único con eventos de dominio.

Order Commands
 - CancelOrder
 - CreateOrder
 - CreateOrderDraft
 - ShipOrder

Order Domain Events
 - BuyerAndPaymentMethodVerified
 - OrderCancelled
 - OrderGracePeriodConfirmed
 - OrderPaid
 - OrderShipped
 - OrderStarted
 - OrderStockConfirmed


# Complejidad del modelo de dominio
A la hora de realizar el diseño de un servicio, se crea un modelo específico para él. Usar comunicación basada por eventos nos permite usar diferentes arquitecturas internas en cada servicio.

Algunas de las diferentes opciones son:
 * Simple CRUD, single-tier, single-layer.
 * Traditional N-Layered.
 * Domain-Driven Design N-layered.
 * Clean Architecture.
 * Command and Query Responsibility Segregation (CQRS).
 * Event-Driven Architecture (EDA).

La aplicación eShopOnContainers hace uso de diferentes arquitecturas de acuerdo al nivel de complejidad de cada servicio.
 * Los servicios Catalog y Basket usan una arquitectura CRUD simple de una sola capa
 * El servicio de Orders usa una arquitectura Domain-Driven Design

Después de hacer una versión simplificada del servicio Catalog usando NServiceBus como bus de eventos, hice una versión del mismo servicio usando CQRS y DDD con un framework llamado EventFlow. Aunque tuve una buena experiencia con este framework, me parece que tratar de usar estos patrones en un servicio que no tiene la suficiente complejidad como para justificarlos acaba creando un servicio innecesariamente complejo. Usar una arquitectura de microservicios conlleva una complejidad extra que anteriormente no se tenía. Manejar esa complejidad a un nivel aceptable debe ser un asunto de prioridad.

Usar un modelo de dominio en un microservicio permite reducir la brecha entre el negocio como existe y el diseño que se hace para resolver su problema. El servicio Catalog, como está implementado de acuerdo al libro, no contiene suficiente lógica o entidades de negocio como para justificar reescribirlo completamente usando los patrones DDD. Dado que el proyecto está diseñado con fines didácticos, usa un dominio simplificado. El manejo de existencias en el inventario es un atributo en la entidad CatalogItem, que se reduce en el EventHandler del evento OrderStatusChangedToPaid.

# Propuesta de implementación
Dado que los eventos de integración deben manejarse mediante un bus de servicio que permita publicarlos asíncronamente, debemos analizar si los medios que usamos actualmente tienen la funcionalidad necesaria.

Una cosa que he notado en varios frameworks de CQRS es que no hacen una distinción entre eventos de integración y eventos de dominio. 

En el caso de EventFlow, aunque tiene soporte para publicar los eventos en un bus con RabbitMQ, me parece que le hace falta una funcionalidad más robusta para poder usarlo en un ambiente de producción. La opción que estoy considerando es usar EventFlow solo para el manejo de eventos de dominio y usar NServiceBus o MassTransit para el manejo de eventos de integración. De este modo, solo es necesario usar EventFlow en los servicios que tengan un modelo de dominio DDD, mientras que en los servicios CRUD solo tenemos que implementar los eventos de integración junto con sus handlers.

Si se prefiere no usar un bus de servicio separado y el framework usado no hace distinción entre eventos de dominio y eventos de integración (como me parece que es el caso con Eventuate), una posible opción sería usar un servicio de integración. El servicio sería una clase dentro del microservicio que pueda manejar eventos de integración publicados en el bus. El punto clave de esta implementación es tener un lugar donde procesar eventos que no esté dentro de un aggregate.


# Referencias
 * .NET Microservices Architecture for Containerized .NET Applications: 
 * .NET Microservices Architecture for Containerized .NET Applications: Domain events: design and implementation (Pag 235)
 * Design a microservice-oriented application | Microsoft Docs (https://docs.microsoft.com/en-us/dotnet/standard/microservices-architecture/multi-container-microservice-net-applications/microservice-application-design)