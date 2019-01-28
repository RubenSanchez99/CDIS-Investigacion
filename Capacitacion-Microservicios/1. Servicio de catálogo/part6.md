# Capacitación Microservicios: Módulo 1 - Config, extensions y ViewModel

Podemos acceder a al objeto IConfiguration en cualquier clase del proyecto mediante inyencción de dependencias, pero también podemos crear una clase que contenga los datos de configuración que nos sean relevantes, de modo que no se tenga que acceder a IConfiguration como si fuera un diccionario.

Se agrega una clase Settings que tenga como propiedades las configuraciones que queremos usar.

```csharp
namespace Catalog.API
{
    public class CatalogSettings
    {
        public string PicBaseUrl { get;set;}
    }
}
```

Agregamos la clase como servicio en Startup.

```csharp
namespace Catalog.API
{
    public class Startup
    {
        public IConfiguration Configuration { get; }

        public IServiceProvider ConfigureServices(IServiceCollection services)
        {
            services.Configure<CatalogSettings>(Configuration);
        }
    }
}
```

Usamos la clase Settings mediante inyección de dependencias.

```csharp
namespace Catalog.API.Controllers
{
    [Route("api/v1/[controller]")]
    public class CatalogController : ControllerBase
    {
        private readonly CatalogSettings _settings;

        public CatalogController(IOptionsSnapshot<CatalogSettings> settings)
        {
            _settings = settings.Value;
        }
    }
}
```

Un método de extensión es aquel que se puede agregar a una clase sin modificarla. El siguiente método FillProductUrl se puede usar sobre cualquier objeto CatalogItem (que aparece en los parámetros del constructor como 'this'), siempre que se haya agregado el namespace de la clase de extensión con 'using'.

```csharp
namespace Catalog.API.Model
{
    public static class CatalogItemExtensions
    {
        public static void FillProductUrl(this CatalogItem item, string picBaseUrl)
        {
            if (item != null)
            {
                item.PictureUri = picBaseUrl.Replace("[0]", item.Id.ToString());
            }
        }
    }
}
```

Un ViewModel es una clase que solo se usa para retornar datos para una vista. Esta clase hace uso de generics, de modo que podamos usarla para encapsular una lista de entidades de cualquier tipo. Al usar esta clase, se reemplaza TEntity por el tipo de entidad que estamos encapsulando.

```csharp
namespace Catalog.API.ViewModel
{
    using System.Collections.Generic;

    public class PaginatedItemsViewModel<TEntity> where TEntity : class
    {
        public int PageIndex { get; private set; }

        public int PageSize { get; private set; }

        public long Count { get; private set; }

        public IEnumerable<TEntity> Data { get; private set; }

        public PaginatedItemsViewModel(int pageIndex, int pageSize, long count, IEnumerable<TEntity> data)
        {
            this.PageIndex = pageIndex;
            this.PageSize = pageSize;
            this.Count = count;
            this.Data = data;
        }
    }
}
```

## Ejercicio

Agregar clases CatalogSettings, CatalogItemExtensios y PaginatedItemsViewModel