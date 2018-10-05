## Servicio de catálogo

## Servicio de carrito de compras

### Repository
El servicio de carrito de compras usa el patrón Repository para no mantener una dependencia con la persistencia mediante Redis, siguiendo el principio de ignorancia de persistencia.

## Servicio de ordenes

### Domain-Driven Design Layers
El servicio de ordenes está estructurado de la siguiente manera:
 * Ordering.API corresponde a a la capa de aplicación
 * Ordering.Domain corresponde a la capa de modelo de dominio
 * Ordering.Infrastructure corresponde a la capa de infraestructura.

### Entities
Las clases Buyer, PaymentMethod, Order y OrderItems son entidades.

### Value Objects
La clase Address del OrderAggregate es un objeto valor.

### Aggregates
El servicio de ordenes tiene dos aggregates:
 * Buyer
    * Buyer
    * PaymentMethods
       * CardType
 * Order
    * Order
    * Address
    * OrderStatus
    * OrderItems

#### Aggregate root
El aggregate root de OrderAggregate es la entidad Order.

El aggregate root de BuyerAggregate es la entidad Buyer.

### Repositories
El servicio de ordenes tiene un repository para cada aggregate

### Enumeration
Las clases CardType y OrderStatus son enumeraciones.
