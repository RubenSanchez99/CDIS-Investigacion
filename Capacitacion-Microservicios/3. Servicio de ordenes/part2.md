# Capacitación Microservicios: Módulo 3 - Modelo de dominio

Antes de proseguir, lea las páginas 182-188, 193-203 y 219-221 del libro "Microservices Architecture for Containerized .NET Applications", las páginas vi-12 del libro ["Domain-Driven Design Reference"](http://domainlanguage.com/wp-content/uploads/2016/05/DDD_Reference_2015-03.pdf) y los siguientes artículos:

* [An In-Depth Look at CQRS](https://blog.sapiensworks.com/post/2015/09/01/In-Depth-CQRS)
* [DDD Decoded - Entities and Value Objects Explained](http://blog.sapiensworks.com/post/2016/07/29/DDD-Entities-Value-Objects-Explained)

Las tres siguientes partes del módulo se enfocarán en la creación de un modelo de dominio usando los patrones vistos en los documentos anteriores.

Para implementar estos patrones de CQRS y DDD, usaremos la librería [EventFlow](https://github.com/eventflow/EventFlow).

El primer paso es crear las clases de entidades, objetos valor y enumeraciones con las que conformaremos los aggregates.

Las clases del modelo de dominio se ubican en la capa Domain, en una carpeta llamada AggregatesModel. Adentro, se hacen carpetas para ubicar las clases de cada aggregate.

Para crear un objeto valor, se crea una clase que herede de ValueObject.

```csharp
using EventFlow.ValueObjects;

namespace Ordering.Domain.AggregatesModel.OrderAggregate
{
    public class Address : ValueObject
    {
        public String Street { get; }
        public String City { get; }
        public String State { get; }
        public String Country { get; }
        public String ZipCode { get; }

        private Address() { }

        public Address(string street, string city, string state, string country, string zipcode)
        {
            Street = street;
            City = city;
            State = state;
            Country = country;
            ZipCode = zipcode;
        }
    }
}
```

Para crear una enumeración, se crea una clase que hereda de Enumeration. La clase Enumeration no es parte de una librería, sino que está ubicada en la carpeta SeedWork de la capa Domain.

```csharp
namespace Ordering.Domain.AggregatesModel.OrderAggregate
{
    public class OrderStatus : Enumeration
    {
        public static OrderStatus Submitted = new OrderStatus(1, nameof(Submitted).ToLowerInvariant());
        public static OrderStatus AwaitingValidation = new OrderStatus(2, nameof(AwaitingValidation).ToLowerInvariant());
        public static OrderStatus StockConfirmed = new OrderStatus(3, nameof(StockConfirmed).ToLowerInvariant());
        public static OrderStatus Paid = new OrderStatus(4, nameof(Paid).ToLowerInvariant());
        public static OrderStatus Shipped = new OrderStatus(5, nameof(Shipped).ToLowerInvariant());
        public static OrderStatus Cancelled = new OrderStatus(6, nameof(Cancelled).ToLowerInvariant());

        protected OrderStatus()
        {
        }

        public OrderStatus(int id, string name)
            : base(id, name)
        {
        }

        public static IEnumerable<OrderStatus> List() =>
            new[] { Submitted, AwaitingValidation, StockConfirmed, Paid, Shipped, Cancelled };
    }
}
```

Para crear una entidad, primero debe crearse una clase que represente su ID. La clase debe heredar de Identity\<T>, donde T es la clase del ID.

```csharp
using System;
using EventFlow.Core;
using EventFlow.ValueObjects;
using Newtonsoft.Json;

namespace Ordering.Domain.AggregatesModel.BuyerAggregate.Identity
{
    [JsonConverter(typeof(SingleValueObjectConverter))]
    public class PaymentMethodId : Identity<PaymentMethodId>
    {
        public PaymentMethodId(string value) : base(value)
        {
        }
    }
}
```

La clase entidad debe heredar de Entity\<T>, donde T es la clase de identidad que le corresponde. El constructor debe tener como primer argumento el id de la entidad, y debe ejecutar el constructor base.

```csharp
namespace Ordering.Domain.AggregatesModel.BuyerAggregate
{
    public class PaymentMethod : Entity<PaymentMethodId>
    {
        private string _alias;
        private string _cardNumber;
        private string _securityNumber;
        private string _cardHolderName;
        private DateTime _expiration;

        private int _cardTypeId;
        public CardType CardType { get; private set; }

        public PaymentMethod(PaymentMethodId id) : base(id)
        {

        }

        public PaymentMethod(PaymentMethodId id, int cardTypeId, string alias, string cardNumber, string securityNumber, string cardHolderName, DateTime expiration) : base(id)
        {
            _cardNumber = cardNumber;
            _securityNumber = securityNumber;
            _cardHolderName = cardHolderName;
            _alias = alias;
            _expiration = expiration;
            _cardTypeId = cardTypeId;
        }
    }
}
```

## Ejercicio

Agregue las clases de OrderAggregate.

* OrderItemId
* Address (ValueObject)
  * String Street
  * String City
  * String State
  * String Country
  * String ZipCode
* OrderItem (Entity)
  * String ProductName
  * string PictureUrl
  * decimal UnitPrice
  * decimal Discount
  * int Units
  * int ProductId
  * Constructor: Lanza OrderingDomainException si units es menor a cero o si (unitPrice * items) es mayor a discount. Inicializa los atribuos del objeto.
  * SetNewDiscount: Recibe un decimal que define el nuevo descuento. Lanza OrderingDomainException si el descuento a asignar es menor a cero.
  * AddUnits: Agrega las unidades recibidas como parámetro. Lanza OrderingDomainException si las unidades a agregar son menores a cero.
* OrderStatus (Enumeration)
  * static OrderStatus Submitted
  * static OrderStatus AwaitingValidation
  * static OrderStatus StockConfirmed
  * static OrderStatus Paid
  * static OrderStatus Shipped
  * static OrderStatus Cancelled
  * protected OrderStatus(): Contructor protegido sin parámetros
  * OrderStatus(int id, string name) : base(id, name)
  * static IEnumerable\<OrderStatus> List(): Retorna un arreglo con todos los posibles estatus
  * static OrderStatus FromName(string name): Recibe el nombre de estatus y retorna un objeto OrderStatus representando el estatus correspondiente. Lanza OrderingDomainException si name no corresponde a ningún estatus.
  * static OrderStatus From(int id): Recibe el id de un estatus y retorna un objeto OrderStatus representando el estatus correspondiente. Lanza OrderingDomainException si name no corresponde a ningún estatus.