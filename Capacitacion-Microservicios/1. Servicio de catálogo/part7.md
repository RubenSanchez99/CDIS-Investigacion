# Capacitación Microservicios: Módulo 1 - Controlador de la API

El [controlador de la API](https://docs.microsoft.com/en-us/aspnet/core/web-api/?view=aspnetcore-2.1) define las rutas que aceptará el microservicio.

Los controladores van en la carpeta Controllers. Cuando usamos un proyecto Web API, el controller hereda de la clase ControllerBase.

El atributo Route define la ruta que aceptará el controlador. En este caso, la ruta base será "api/v1/catalog".

Para definir qué rutas aceptará este controlador, creamos un ActionMethod que responda a una consulta. El atributo HttpGet significa que este ActionMethod responderá a peticiones GET con la ruta especificada en Route. En este ejemplo, el ActionMethod responderá a peticiones GET con la ruta api/v1/catalog/items.

El ActionMethod retorna un [ActionResult](https://docs.microsoft.com/en-us/aspnet/core/web-api/action-return-types?view=aspnetcore-2.2), que es un objeto que encapsula la respuesta de la petición con un código de respuesta de HTTP. En este ejemplo, la línea ```return Ok(model)``` significa que se va a retornar una el objeto model (que es de tipo PaginatedItemsViewModel\<CatalogItem>) con un código de respuesta 200 OK.

Este ActionMethod se define como un método [async](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/async/). Los métodos async realizan operaciones asíncronas en su interior. Un método async retorna un Task, que es una representación de una tarea asíncrona. Para ejecutar la acción asíncronamente, se usa el comando ```await```. El comando await se usa al ejecutar un método que retorne un Task y hace que retorne el objeto que encapsula, una vez que termine su ejecución.

Los datos que recibe el ActionMethod se configuran como parámetros. Las etiquetas \[FromX] definen de donde se sacarán los datos.

* \[FromBody], del cuerpo de la petición (generalmente en JSON)
* \[FromForm], de un POST de un form de HTML
* \[FromHeader], de los headers de la petición
* \[FromQuery], del querystring de la petición (las variables que vienen con formato "?key=value en la dirección de la petición)

Las consultas se hacen sobre la clase CatalogContext, que se obtiene mediante inyección de dependencias. Se usa [LINQ to Entities](https://docs.microsoft.com/en-us/dotnet/csharp/tutorials/working-with-linq) para hacer las consultas hacia la base de datos.

```csharp
namespace Catalog.API.Controllers
{
    [Route("api/v1/[controller]")]
    public class CatalogController : ControllerBase
    {
        // GET api/v1/[controller]/items[?pageSize=3&pageIndex=10]
        [HttpGet]
        [Route("[action]")]
        public async Task<IActionResult> Items([FromQuery]int pageSize = 10, [FromQuery]int pageIndex = 0)
        {
            var totalItems = await _catalogContext.CatalogItems
                .LongCountAsync();

            var itemsOnPage = await _catalogContext.CatalogItems
                .OrderBy(c => c.Name)
                .Skip(pageSize * pageIndex)
                .Take(pageSize)
                .ToListAsync();

            var model = new PaginatedItemsViewModel<CatalogItem>(
                pageIndex, pageSize, totalItems, itemsOnPage);

            return Ok(model);
        }
    }
}
```

Para agregar registros a las tablas, se accede a la propiedad del CatalogContext que corresponde a la entidad a agregar (en este caso, CatalogItems) y se usa el método Add. Los cambios a registros de las tablas no se efectúan hasta que se ejecuta el método SaveChangesAsync() (o su versión síncrona, SaveChanges()).

```csharp
        //POST api/v1/[controller]/items
        [Route("items")]
        [HttpPost]
        public async Task<IActionResult> CreateProduct([FromBody]CatalogItem product)
        {
            var item = new CatalogItem
            {
                CatalogBrandId = product.CatalogBrandId,
                CatalogTypeId = product.CatalogTypeId,
                Description = product.Description,
                Name = product.Name,
                Price = product.Price
            };
            _catalogContext.CatalogItems.Add(item);

            await _catalogContext.SaveChangesAsync();

            return CreatedAtAction(nameof(GetItemById), new { id = item.Id }, null);
        }
```

## Ejercicio

Crear CatalogController y agregar los siguientes métodos:

* Items: Acepta la ruta "GET api/v1/[controller]/items[?pageSize=3&pageIndex=10]" y retorna todos los CatalogItems de la base de datos en un PaginatedItemsViewModel. Usar los valores de pageSize y pageIndex para paginado.
* CreateProduct: Acepta la ruta "POST api/v1/[controller]/items" y un \[FromBody]CatalogItem. Agrega un CatalogItem a la base de datos.
* UpdateProduct: Acepta la ruta "PUT api/v1/[controller]/items" y un \[FromBody]CatalogItem. Actualiza un CatalogItem. Retorna un NotFound si se envía un Id inexistente.
* DeleteProduct: Acepta la ruta "DELETE api/v1/[controller]/id". Elimina el CatalogItem dado como parámetro.