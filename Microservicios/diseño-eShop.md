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
## Redirecciones

# API Gateway Aggregator
## Descripción
## Servicios
## API