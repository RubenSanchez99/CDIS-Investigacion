# Servicio Catalog
## Descripción
Este servicio guarda la información de los productos de la tienda, junto con su disponibilidad de inventario.

## Modelo


## Eventos de integración
Eventos publicados por el servicio:
  - ProductPriceChangedIntegrationEvent
  - OrderStockRejectedIntegrationEvent
  - OrderStockConfirmedIntegrationEvent

Eventos a los que se suscribe:
  - OrderStatusChangedToAwaitingValidationIntegrationEvent
    * Handler: Publica el evento OrderStockConfirmed si hay disponibilidad de inventario en todos los productos de la orden, o el evento OrderStockRejected si hay un producto sin disponibilidad 
  - OrderStatusChangedToPaidIntegrationEvent
    * Handler: Reduce el número de stock de los productos de la orden
## Servicios
ICatalogIntegrationService (solo si no se usa un bus de servicio dedicado)

## API
http://187.162.37.224:5101/swagger/

# Servicio Basket
## Descripción
Este servicio almacena en un caché de Redis los productos del carrito de compras de cada usuario

## Autenticación
  * Authority: URL del servicio de identidad
  * Audience: basket

## Modelo
BasketCheckout
 * string City
 * string Street
 * string State 
 * string Country
 * string ZipCode
 * string CardNumber
 * string CardHolderName
 * DateTime CardExpiration
 * string CardSecurityNumber
 * int CardTypeId
 * string Buyer
 * Guid RequestId

BasketItem
 * string Id
 * string ProductId
 * string ProductName
 * decimal UnitPrice
 * decimal OldUnitPrice
 * int Quantity 
 * string PictureUrl
 * Validate(): Valida que Quantity sea por lo menos 1

CustomerBasket
  * string BuyerId
  * List<BasketItem> Items

## Repositorios
IBasketRepository
  * CustomerBasket GetBasketAsync(string customerId);
  * List<string> GetUsers();
  * CustomerBasket UpdateBasketAsync(CustomerBasket basket);
  * bool DeleteBasketAsync(string id);

RedisBasketRepository
  * CustomerBasket GetBasketAsync(string customerId);
  * List<string> GetUsers();
  * CustomerBasket UpdateBasketAsync(CustomerBasket basket);
  * bool DeleteBasketAsync(string id);
  * GetDatabase();
    * Retorna el objeto de conexión con la base de datos, o manda a llamar al método ConnectToRedisAsync, de no existir
  * GetServer();
    * Retorna el server del primer endpoint de la conexión con Redis
  * ConnectToRedisAsync();
    * Realiza la conexión con la base de datos

El basket se persiste de la siguiente manera:
  * Key: BuyerId
  * Value: CustomerBasket serializado como JSON

## Eventos de integración
Eventos publicados por el servicio:
  - UserCheckoutAcceptedIntegrationEvent

Eventos a los que se suscribe:
  - OrderStartedIntegrationEvent
    * Handler: Elimina el basket del repositorio
  - ProductPriceChangedIntegrationEvent
    * Handler: Cambia el precio de todos los productos guardados

## Servicios
IdentityService
  * GetUserIdentity: Obtiene el atributo sub del token de identificación

## API
http://187.162.37.224:5102/swagger/

BasketController
  * GET api/v1/basket/{id}
    * Retorna un CustomerBasket correspondiente al id del comprador. De no existir, retorna un basket vacío.
  * POST api/v1/basket
    * Recibe los datos de un CustomerBasket y guarda los cambios mediante el repository
  * POST api/v1/basket/checkout
    * Recibe un BasketCheckout y un requestId. Valida que exista el basket y publica el evento UserCheckoutAccepted
  * DELETE api/v1/basket/{id}
    * Elimina el basket con el id de comprador dado mediante el repositorio

# API Gateway
## Descripción
Expone una API pública que actúa como intermediario entre la interfaz y las APIs de los microservicios
## Redirecciones
Upstream
  * Template: "/api/{version}/c/{everything}"
  * HttpMethods: GET
Downstream
  * Template: "/api/{version}/{everything}"
  * Scheme: http
  * Host: catalog.api
  * Port: 80

Upstream
  * Template: "/api/{version}/b/{everything}"
  * HttpMethods: 
  * AuthenticationProviderKey: "IdentityApiKey"
Downstream
  * Template: "/api/{version}/{everything}"
  * Scheme: http
  * Host: basket.api
  * Port: 80

Upstream
  * Template: "/{everything}"
  * HttpMethods: "POST", "PUT", "GET" 
  * AuthenticationProviderKey: "IdentityApiKey"
Downstream
  * Template: "/{everything}"
  * Scheme: http
  * Host: webshoppingagg
  * Port: 80

# API Gateway Aggregator
## Descripción
El aggregator recibe peticiones redirigidas por el Gateway que necesitan consultar información de uno o más microservicios. Agrega el bearer token a las peticiones hechas a los servicios Catalog, Basket y Ordering. 

## Políticas de reintentos
Catalog Service, Basket Service y Ordering Service
  * Wait and Retry: Reintentar 6 veces, con un espacio de 2^n entre cada intento (n = numero de reintento actual)
  * Circuit Breaker: Dejar de intentar por 30 segundos después de 5 intentos fallidos

## Autenticación
  * Authority: URL del servicio de identidad
  * Audience: webshoppingagg

## Modelo
AddBasketItemRequest
  * string CatalogItemId
  * string BasketId
  * int Quantity = 1

BasketData
  * string BuyerId 
  * List<BasketDataItem> Items = new List<BasketDataItems>()

BasketDataItem
  * string Id 
  * string ProductId
  * string ProductName
  * decimal UnitPrice
  * decimal OldUnitPrice 
  * int Quantity 
  * string PictureUrl

CatalogItem
  * int Id
  * string Name
  * decimal Price
  * string PictureUri

UpdateBasketItemsRequest
  * string BasketId
  * ICollection<UpdateBasketItemData> Updates

UpdateBasketItemData
  * string BasketItemId
  * int NewQty

UpdateBasketRequest
  * string BuyerId
  * IEnumerable<UpdateBasketRequestItemData> Items

UpdateBasketRequestItemData
  * string Id
  * int ProductId
  * int Quantity

## Servicios
CatalogService
  * GetCatalogItem(string id)
    * Consulta en el microservicio Catalog el producto con el Id dado. Retorna un CatalogItem
  * GetCatalogItems(List<int> ids) 
    * Consulta en el microservicio Catalog el producto con los Id dados. Retorna una lista de CatalogItem
BasketService
  * GetById(string id)
    * Consulta al microservicio Basket para obtener la información del carrito de compras. Retorna un BasketData
  * Update(BasketData currentBasket)
    * Envía el contenido del currentBasket a la ruta para actualizar del microservicio Basket
## API
BasketController
  * POST/PUT api/v1/basket (UpdateAllBasket)
    * Recibe un UpdateBasketRequest, obtiene la información de los productos del servicio de Catalogo, los agrega a un BasketData como BasketDataItem y ejecuta el método Update del BasketService el BasketData como parámetro
  * PUT api/v1/basket/items (UpdateQuantities)
    * Recibe un UpdateBasketItemsRequest, consulta el basket con el ID dado, consulta cada producto actualizado y actualiza su producto equivalente en el basket
  * POST api/v1/basket/items (AddBasketItem)
    * Recibe un AddBasketItemRequest, obtiene la información del producto del CatalogService, crea u obtiene el basket mediante el BasketService, agrega el producto al basket y lo actuliza mediante el método Update de BasketService

# Servicio Ordering
## Descripción
Este servicio maneja el flujo de órdenes.

IntegrationEventHandler -> Método del Aggregate -> DomainEvent
CommandHandler -> Métodod del Aggregate -> DomainEvent

## Modelo
### OrderAggregate
Atributos
  * private DateTime _orderDate
  * public Address Address
  * private _buyerId;
  * public OrderStatus OrderStatus
  * private string _description
  * private bool _isDraft
  * private List<OrderItems> _orderItems
  * private Nullable<int> _paymentMethodId

Métodos
  * Order
    * Inicializa los datos y publica el evento de dominio OrderStarted
  * AddOrderItem
    * Agrega un producto al la lista de productos
  * SetPaymentId
  * SetBuyerId
  * SetAwaitingValidationStatus
    * Valida que la orden esté en estatus Submitted y publica el evento de dominio OrderStatusChangedToAwaitingValidation
    * Cambia el estatus a AwaitingValidation
  * SetStockConfirmedStatus
    * Valida que la orden esté en estatus AwaitingValida## Flujo principal
  * Se recibe el comando tion y publica el evento de dominio OrderStatusChangedToStockConfirmed
    * Cambia el estatus a StockConfirmed
  * SetPaidStatus
    * Valida que la orden esté en estatus StockConfirmed y publica el evento de dominio OrderStatusChangedToPaid
    * Cambia el estatus a Paid
  * SetShippedStatus
    * Valida que la orden esté en estatus Paid y publica el evento de dominio OrderShipped
    * Cambia el estatus a Shipped
  * SetCancelledStatus
    * Valida que la orden no esté en estatus Paid o Shipped y publica el evento de dominio OrderCancelled
  * SetCancelledStatusWhenStockIsRejected
    * Si la orden está en estatus AwaitingValidation, cambia el estatus a Cancelled y pone en la descripción un mensaje con los productos que no tenían suficiente inventario
  * AddOrderStartedDomainEvent
    * Publica el evento de dominio OrderStarted

### OrderStatus
Enum:
  * 1: Submitted
  * 2: AwaitingValidation
  * 3: StockConfirmed
  * 4: Paid
  * 5: Shipped
  * 6: Cancelled

### Address
Value Object

Atributos:
  * string Street
  * string City
  * string State
  * string Country
  * string ZipCode

### OrderItem
Entity

Atributos:
  * string _productName
  * string _pictureUrl
  * decimal _unitPrice
  * decimal _discount
  * int _units
  * ProductId

Metodos
  * Constructor(int productId, string productName, decimal unitPrice, decimal discount, string PictureUrl, int units = 1)
    * Lanza una OrderingDomainException si units es menor o igual a 0, con el mensaje "Invalid number of units"
    * Lanza una OrderingDomainException si (unitPrice * units) es menor que discount, con el mensaje "The total of order item is lower than applied discount"
  * SetNewDiscount(decimal discount)
    * Lanza una OrderingDomainException si discount es menor que 0, con el mensaje "Discount is not valid"
  * AddUnits(int units)
    * Lanza una OrderingDomainException si units es menor que 0, con el mensaje "Invalid units"

### BuyerAggregate
Atributos
  * string IdentityGuid
  * string Name
  * List<PaymentMethod> _paymentMethods

Métodos
  * Constructor (string identity, string name)
    * Valida que identity y name no estén vacios
  * PaymentMethod VerifyOrAddPaymentMethod(
    int cardTypeId, string alias, string cardNumber, 
    string securityNumber, string cardHolderName, DateTime expiration, int orderId)
    * Si el Buyer tiene un PaymentMethod con los datos cardTypeId, cardNumber y expiration, se genera el evento de dominio BuyerAndPaymentMethodVerifiedDomainEvent
    * Si el Buyer no tiene un PaymentMethod con esos datos, se crea uno nuevo y se agrega a la lista _paymentMethods. Se genera el evento de dominio BuyerAndPaymentMethodVerifiedDomainEvent

### PaymentMethod
Entity

Atributos
  * string _alias
  * string _cardNumber
  * string _securityNumber
  * string _cardHolderName
  * DateTime _expiration
  * CardType CardType

Métodos
  * Constructor(int cardTypeId, string alias, string cardNumber, string securityNumber, string cardHolderName, DateTime expiration)
    * Valida que cardNumber, securityNumber y cardHolderName no estén vacios
    * Si la fecha expiration es menor a la fecha de hoy, se genera una OrderingDomainException
  * bool IsEqualTo(int cardTypeId, string cardNumber,DateTime expiration)
    * Valida que los datos de este PaymentMethod correspondan a los datos dados como parámetro

### CardType
Enum:
  * 1: Amex
  * 2: Visa
  * 3: MasterCard

## Comandos
 - CancelOrder
   * Handler: Ejecuta el método SetCancelledStatus de la orden
 - CreateOrder
   * Handler: Crea una orden nueva y añade los artículos mediante el método AddOrderItem 
 - CreateOrderDraft
   * Handler: Crea una orden nueva, la convierte a un OrderDraftDTO y lo retorna
 - ShipOrder
   * Handler: Ejecuta el método SetShippedStatus de la orden

## Eventos de dominio
 - BuyerAndPaymentMethodVerified
 - OrderCancelled
 - OrderGracePeriodConfirmed
 - OrderPaid
 - OrderShipped
 - OrderStarted
 - OrderStockConfirmed

## Comportamientos de eventos de dominio
BuyerAndPaymentMethodVerified
  * UpdateOrderWhenBuyerAndPaymentMethodVerifiedDomainEventHandler (BuyerAggregate)
    * When the Buyer and Buyer's payment method have been created or verified that they existed, then we can update the original Order with the BuyerId and PaymentId (foreign keys)

OrderCancelled
  * OrderCancelledDomainEventHandler Subscriber
    * Publica el evento OrderStatusChangedToCancelledIntegrationEvent

OrderGracePeriodConfirmed
  * OrderStatusChangedToAwaitingValidationDomainEventHandler (Subscriber)
    * Publica el evento OrderStatusChangedToAwaitingValidationIntegrationEvent

OrderPaid
  * OrderStatusChangedToPaidDomainEventHandler (Subscriber)
    * Publica el evento de integración OrderStatusChangedToPaidIntegrationEvent

OrderShipped
  * OrderShippedDomainEventHandler (Subscriber)
    * Publica el evento de integración OrderStatusChangedToShippedIntegrationEvent

OrderStarted
  * ValidateOrAddBuyerAggregateWhenOrderStartedDomainEventHandler (BuyerAggregate)
    * Revisa si la orden ya tiene un Buyer asignado. Si no lo tiene, crea un nuevo Buyer con los datos del usuario recibidos en el evento. Si no había un Buyer con el id del usuario, lo agrega al repositorio. Si ya había uno con el id del usuario, lo actualiza.
    * Ejecuta el método VerifyOrAddPaymentMethod del Buyer

OrderStockConfirmed
  * OrderStatusChangedToStockConfirmedDomainEventHandler
    * Publica el evento de integración OrderStatusChangedToStockConfirmedIntegrationEvent

## Servicios de dominio
## Eventos de integración
Eventos publicados por el servicio:
  - OrderStatusChangedToCancelled
  - OrderStatusChangedToAwaitingValidation
  - OrderStatusChangedToPaid
  - OrderStatusChangedToShipped
  - OrderStatusChangedToSubmitted
  - OrderStatusChangedToStockConfirmed

Eventos a los que se suscribe:
  - GracePeriodConfirmed
    * Handler: Indica que terminó el periodo de espera de la orden y cambia su estado a pendiente de validar.
  - OrderPaymentFailed
    * Handler: Cambia el estatus de la orden a cancelado.
  - OrderPaymentSucceded
    * Handler: Cambia el estatus de la orden a pagado
  - OrderStockConfirmed
    * Handler: Cambia el estatus de la orden a confirmada disponibilidad de inventario
  - OrderStockRejected
    * Handler: Cambia el estatus de la orden a cancelado por falta de inventario.
  - UserCheckoutAccepted
    * Handler: Lanza el comando CreateOrder

## Sagas
### GracePeriod
Cuando se crea una orden, se inicia un GracePeriod. El GracePeriod consiste en un periodo de tiempo donde se realiza la validación de existencia de inventario y el método de pago. 

Datos de estado de la saga:
  * int OrderIdentifier 
  * string UserId 
  * bool GracePeriodIsOver 
  * bool StockConfirmed 

La saga inicia cuando se publica el evento OrderStartedIntegrationEvent. Cuando se recibe ese evento, se iniciarán tres procesos asíncronamente:
  * Iniciar el grace period (Dentro del servicio Ordering)
  * Verificar el método de pago (Dentro del servicio Payments)
  * Verificar existencias de inventario (Dentro del servicio Catalog)

La saga publica el evento OrderStatusChangedToAwaitingValidationIntegrationEvent. El servicio Catalog está suscritos a ese evento. La saga también inicia un timeout, después del cual se publicará el evento GracePeriodExpired.

Cuando se recibe el evento GracePeriodExpired, se cambia la propediad GracePeriodIsOver del estado de la saga a verdadero y se ejecuta el método privado ContinueOrderingProcess.

El método ContinueOrderingProcess valida que los datos GracePeriodIsOver y StockConfirmed sean verdaderos. Si lo son, se pubica el evento OrderStatusChangedToStockConfirmedIntegrationEvent. El servico Payments está suscrito a ese evento.

La saga espera dos eventos generados por el servicio Catalog después del evento OrderStatusChangedToAwaitingValidationIntegrationEvent:
  * OrderStockConfirmedIntegrationEvent: Cambia el estatus StockConfirmed a verdadero
  * OrderStockRejectedIntegrationEvent: Termina la saga

La saga espera dos eventos generados por el servicio Payments después del evento OrderStatusChangedToStockConfirmedIntegrationEvent:
  * OrderPaymentSuccededIntegrationEvent: Termina la saga
  * OrderPaymentFailedIntegrationEvent: Termina la saga

Si se recibe el evento OrderCancelledIntegrationEvent, la saga se termina.

## Consultas
  * GetOrderAsync
  * GetOrdersAsync
  * GetCardTypesAsync

## Servicios
## API
OrdersController
  * PUT api/v1/orders/cancel CancelOrder
    * Publica el comando CancelOrder
  * PUT api/v1/orders/ship ShipOrder 
    * Publica el comando ShipOrder
  * POST api/v1/orders/draft GetOrderDraftFromBasketData
    * Ejecuta el comando CreateOrderDraft
  * GET api/v1/orders/{orderId} GetOrder
    * Ejecuta el query GetOrderAsync
  * GET api/v1/orders GetOrders
    * Ejecuta el query GetOrdersAsync
  * GET api/v1/orders/cardtypes GetCardTypes
    * Ejecuta el query GetCardTypesAsync

# Servicio Payment
## Descripción
## Modelo
## Eventos de integración
Eventos publicados por el servicio:
  - OrderPaymentSucceded
  - OrderPaymentFailed

Eventos a los que se suscribe:
  - OrderStatusChangedToStockConfirmed
    * Handler: Publica eventos OrderPaymentSucceded u OrderPaymentFailed, dependiendo del resultado del pago
## Servicios
## API

# Servicio Identity
## Descripción
Este servicio maneja la parte de autenticación del sistema. Usa el protocolo OpenID Connect.