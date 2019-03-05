# Curso de capacitación de desarrollo de microservicios con ASP.NET Core

El propósito de este curso es capacitar al desarrollador en la creación de microservicios usando ASP.NET Core. 

El proyecto consiste en una tienda donde se pueden comprar diferentes productos. El proyecto completo está compuesto por los siguientes microservicios.

* Catalog: Maneja el catálogo de productos y las existencias de inventario
* Basket: Maneja los carritos de compra de los usuarios
* Ordering: Maneja el procesamiento de órdenes de compra
* Payment: Simula el procesamiento del pago de una órden
* Identity: Maneja la información de los usuarios y los servicios de autenticación
* SignalR Hub: Envía mensajes a la interfaz gráfica para actualizarla en respuesta a eventos
* WebMVC: Interfaz gráfica hecha con ASP.NET MVC

## Módulos del curso

### [Módulo 1: Servicio de catálogo](1.%20Servicio%20de%20catálogo/README.md)

En este módulo se crea un microservicio con funcionalidad CRUD. En el microservicio Catalog se pueden consultar, agregar, actualizar y eliminar productos. El propósito de este módulo es capacitar al desarrollador en la creación de proyectos de ASP.NET Core WebAPI y el uso de Entity Framework Core para la interacción con base de datos. También se ve el uso de Docker para crear contenedores que aislan la funcionalidad de un microservicio y permiten que se genere un componente configurable para entornos de desarrollo y pruebas. Se ve el uso de Git y el ciclo GitFlow para el versionamiento del desarrollo.

### [Módulo 2: Servicio de carrito de compras](2.%20Servicio%20de%20carrito%20de%20compras/README.md)

En este módulo se empieza con un microservicio ya desarollado, el servicio Basket. El servicio Basket también tiene funcionalidad CRUD como el servicio Catalog, usando Redis como almacenamiento en vez de Entity Framework Core. El propósito del módulo es capacitar al desarrollador en la creación de funcionalidad extra como logs, autorización y validaciones del modelo. El módulo presenta dos patrones cuyo propósito es unir la funcionalidad de múltiples microservicios: el API Gateway y el Aggregator. Por último, se ve el uso de eventos de integración, el medio por el cual ocurre la comunicación entre microservicios.

### [Módulo 3: Servicio de ordenes](./3.%20Servicio%20de%20ordenes/README.md)

En este módulo se crea un microservicio con un modelo de dominio siguiendo los patrones del Domain-Driven Design. Se usa el patrón CQRS para separar acciones dirigidas hacia los dos modelos que se crean para este microservicio: el modelo de lectura y el modelo de escritura. Se usa Event Sourcing para persistir el estado de las entidades del modelo de lectura mediante eventos de dominio. Se usa el concepto de sagas para realizar transacciones que involucran múltiples microservicios.

## Requisitos previos

Se recomienda que la capacitación se realice usando una distribución de Linux con el editor de texto Visual Studio Code.

El proyecto a realizar hará uso de contenedores de Docker. Sin embargo, el proyecto se puede ejecutar sin los contenedores en el entorno de desarrollo. La carpeta 'scripts' contiene scripts de bash que se pueden usar para ejecutar el proyecto completo, con o sin Docker.

.NET Core puede compilar el proyecto en Linux, MacOS y Windows. Los pasos especificados en el curso están basados en Linux con la terminal bash.

Es recomendable que haya leído el libro '.NET Microservices: Architecture for Containerized .NET Applications'. Este libro se tomará como referencia durante el transcurso de los módulos.

## Software

```
git 2.18.0
gitflow-avh  1.11.0-1
dotnet-sdk 2.1.2
postman 6.3.0
docker 18.06
docker-compose 22.0
pgAdmin 4.0
```

## Extensiones de VS Code

Se recomienda usar las siguientes extensiones de Visual Studio Code para simplificar la funcionalidad de algunos pasos que se seguirán en el curso.

* C#
* gitflow
* Docker
* C# Extensions
* C# XML Documentation Comments
* Better Comments
* .NET Core Tools
* NuGet Package Manager

## Material extra

* [.NET Microservices: Architecture for Containerized Applications](https://aka.ms/microservicesebook)