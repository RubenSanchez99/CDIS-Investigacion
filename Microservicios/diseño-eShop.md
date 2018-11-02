# Servicio Catalog
## Descripción
Este servicio guarda la información de los productos de la tienda, junto con su disponibilidad de inventario.

## Modelo


## Eventos de integración
Eventos publicados por el servicio:
  - ProductPriceChanged
  - OrderStockRejected
  - OrderStockConfirmed
Eventos a los que se suscribe:
  - OrderStatusChangedToAwaitingValidation
    * Handler: Publica el evento OrderStockConfirmed si hay disponibilidad de inventario en todos los productos de la orden, o el evento OrderStockRejected si hay un producto sin disponibilidad 
  - OrderStatusChangedToPaid
    * Handler: Reduce el número de stock de los productos de la orden
## Servicios
ICatalogIntegrationService (solo si no se usa un bus de servicio dedicado)

## API

# Servicio Basket
## Descripción
Este servicio almacena en un caché de Redis los productos del carrito de compras de cada usuario

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
  * Task<CustomerBasket> GetBasketAsync(string customerId);
  * IEnumerable<string> GetUsers();
  * Task<CustomerBasket> UpdateBasketAsync(CustomerBasket basket);
  * Task<bool> DeleteBasketAsync(string id);
## Eventos de integración
Eventos publicados por el servicio:
  - UserCheckoutAccepted
Eventos a los que se suscribe:
  - OrderStarted
    * Handler: Elimina el basket del repositorio
  - ProductPriceChanged
    * Handler: Cambia el precio de todos los productos guardados
## Servicios
IdentityService
  * GetUserIdentity
## API
BasketController
  * GET api/v1/basket/{id}
    * Retorna un CustomerBasket correspondiente al id del comprador. De no existir, lo crea.
  * POST api/v1/basket
    * Recibe los datos de un CustomerBasket y guarda los cambios mediante el repository
  * POST api/v1/basket/checkout
    * Recibe un BasketCheckout y un requestId. Valida que exista el basket y publica el evento UserCheckoutAccepted
  * DELETE api/v1/basket/{id}
    * Elimina el basket con el id dado mediante el repositorio

# Servicio Ordering
## Descripción
Este servicio maneja el flujo de órdenes.

IntegrationEventHandler -> Método del Aggregate -> DomainEvent
CommandHandler -> Métodod del Aggregate -> DomainEvent

## Modelo
OrderAggregate
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
    * Valida que la orden esté en estatus AwaitingValidation y publica el evento de dominio OrderStatusChangedToStockConfirmed
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

BuyerAggregate

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
## Consultas
## Servicios
## API

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
El aggregator recibe peticiones redirigidas por el Gateway que necesitan consultar información de uno o más microservicios.
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
  * GetCatalogItems(IEnumerable<int> ids) 
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

# Servicio Identity
## Descripción
Este servicio maneja la parte de autenticación del sistema. Usa el protocolo OpenID Connect.