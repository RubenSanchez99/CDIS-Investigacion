# Capacitación Microservicios: Módulo 2 - Aggregator

El Aggregator es un microservicio que se encarga de realizar las peticiones que abarcan múltiples servicios.

Para que una ruta vaya hacia el aggregator, se debe asignar en una redirección dentro del gateway.

```json
{
  "ReRoutes": [
    {
      "DownstreamPathTemplate": "/api/{version}/basket/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "webshoppingagg",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/api/{version}/basket/{everything}",
      "UpstreamHttpMethod": [ "PUT", "POST" ],
      "AuthenticationOptions": {
        "AuthenticationProviderKey": "IdentityApiKey",
        "AllowedScopes": []
      }
    }
  ],
    "GlobalConfiguration": {
      "RequestIdKey": "OcRequestId",
      "AdministrationPath": "/administration"
    }
}
```

El aggregator recibirá la petición redirigida por el gateway y hará peticiones a los servicios involucrados en el proceso. La razón de delegar la lógica que involucra múltiples servicios al gateway (en vez de, por ejemplo, hacer que un microservicio llame directamente a otro) es prevenir dependencias entre los microservicios. Un microservicio debe poder realizar las tareas que se le asignaron sin depender de la API de otros microservicios.

Cuando un microservicio es parte de un proceso que involucre múltiples servicios, la comunicación entre ellos se hará solamente mediante eventos de integración.

Para que un aggregator pueda llamar a un microservicio, debe crearse un HttpClient en la clase Startup.cs usando el método AddHttpClient.

Se crea una interfaz para definir qué llamadas podrán hacerse al microservicio.

```csharp
    public interface ICatalogService
    {
        Task<CatalogItem> GetCatalogItem(int id);
        Task<IEnumerable<CatalogItem>> GetCatalogItems(IEnumerable<int> ids);
    }

```

Se usa una clase UrlsConfig para guardar las rutas hacia las que se harán peticiones en el servicio.

```csharp
    public class UrlsConfig
    {
        public class CatalogOperations
        {
            public static string GetItemById(int id) => $"/api/v1/catalog/items/{id}";
            public static string GetItemsById(IEnumerable<int> ids) => $"/api/v1/catalog/items?ids={string.Join(',', ids)}";
        }

        public string Catalog { get; set; }
    }
```

Se implementa la interfaz del servicio que usaremos para interactuar con el microservicio mediante HttpClient.

```csharp
    public class CatalogService : ICatalogService
    {

        private readonly HttpClient _httpClient;
        private readonly UrlsConfig _urls;

        public CatalogService(HttpClient httpClient, IOptions<UrlsConfig> config)
        {
            _httpClient = httpClient;
            _urls = config.Value;
        }

        public async Task<CatalogItem> GetCatalogItem(int id)
        {
            var stringContent = await _httpClient.GetStringAsync(_urls.Catalog + UrlsConfig.CatalogOperations.GetItemById(id));
            var catalogItem = JsonConvert.DeserializeObject<CatalogItem>(stringContent);

            return catalogItem;
        }

        public async Task<IEnumerable<CatalogItem>> GetCatalogItems(IEnumerable<int> ids)
        {
            var stringContent = await _httpClient.GetStringAsync(_urls.Catalog + UrlsConfig.CatalogOperations.GetItemsById(ids));
            var catalogItems = JsonConvert.DeserializeObject<CatalogItem[]>(stringContent);

            return catalogItems;
        }
    }
```

Dado que es posible que las peticiones hacia el microservicio (ya sea porque está caído o no es accesible), se deben configurar retry policies que determinen qué se debe hacer cuando no se pueda contactar al microservicio.

Se usará la librería [Polly](https://github.com/App-vNext/Polly) para configurar las retry polcies. En la documentación de Polly se pueden ver todas las pólizas que se pueden usar. Las dos que usamos en este proyecto son:

* Retry: Volver a intentar contactar con el microservicio automáticamente. Se puede configurar cuánto tiempo esperar entre cada reintento.
* Cicruit Breaker: Si se sigue reintentando y el microservicio no responde, espera un momento antes de seguir reintentando, y no reintentar hasta que ese periodo de espera haya pasado.

```csharp
        static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
        {
            return HttpPolicyExtensions
              .HandleTransientHttpError()
              .OrResult(msg => msg.StatusCode == System.Net.HttpStatusCode.NotFound)
              .WaitAndRetryAsync(6, retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));
        }

        static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
        {
            return HttpPolicyExtensions
                .HandleTransientHttpError()
                .CircuitBreakerAsync(5, TimeSpan.FromSeconds(30));
        }
```

En el método AddApplicationServices registramos el HttpClient que creamos. Es necesario registrar HttpContext para usar el servicio. HttpClientAuthorizationDelegatingHandler solo se usa en los HttpClient que requieren autorización, ya que contiene la lógica para enviar la nueva petición con el token de autorización.

```csharp
        public static IServiceCollection AddApplicationServices(this IServiceCollection services)
        {
            //register delegating handlers
            services.AddSingleton<IHttpContextAccessor, HttpContextAccessor>();
            services.AddTransient<HttpClientAuthorizationDelegatingHandler>();

            //register http services
            services.AddHttpClient<ICatalogService, CatalogService>()
                //AddHttpMessageHandler solo se agrega si require autorización
                //.AddHttpMessageHandler<HttpClientAuthorizationDelegatingHandler>()
                .AddPolicyHandler(GetRetryPolicy())
                .AddPolicyHandler(GetCircuitBreakerPolicy());

            return services;
        }
```

Una vez registrados todos los services de los microservicios, creamos un controller que reciba las peticiones del cliente. Dentro de los ActionMethods del controlador, se hace uso de los services para consultar o enviar peticiones a los microservicios que participan en el proceso.

## Ejercicio

Agregue la siguiente petición al API Gateway

* PUT y POST para "/api/{version}/basket/{everything}" van dirigdas a webshoppingagg

Agregue el BasketService

Agregue las URL de Basket