# Capacitación Microservicios: Módulo 2 - Filtros

Los filtros de ASP.NET Core son una forma de ejecutar código durante el proceso de procesmiento de una petición recibida por los controladores. Nos son útiles para realizar acciones que se hacen en todas las peticiones.

Existen cinco tipos de filtros:

* Los filtros de autorización (authorization filters) se ejecutan primero y se usan para determinar si el usuario tiene permitido realizar la petición. Pueden terminar la petición en caso de que el usuario no tenga autorización.
* Los filtros de recursos (resource filters)
* Los filtros de acción (action filters) se ejecutan inmediatamente antes y después de llamar un método de acción del controlador. Pueden manipular los argumentos que se pasan al método y también pueden alterar el resultado de la acción.
* Los filtros de excepción (exception filters) se usan para ejecutar código que altere la respuesta de la petición cuando ocurre una excepción no controlada.
* Los filtros de resultado (result filters) se ejecutan solamente cuando la acción terminó exitosamente. Sirven para ingresar lógica relacionada al formato de la respuesta.

Para agregar un filtro, se crea una clase nueva que implemente una de las interfaces del filtro. Se va a agregar el método correspondiente al evento al cual se agrega el código del filtro.

El objeto context tiene información sobre la petición en su estado actual. Ese estado puede leerse y modificarse dentro del filtro.

La clase HttpGlobalExceptionFilter escribe en el log los mensajes de la excepción no controlada, y escribe algo de información en la respuesta de la petición si fue una excepción de dominio. De este modo, podemos recibir los mensajes de error en el front-end.

El filtro también cambia el código de estátus de la respuesta de acuerdo al tipo de excepción. Las excepciones de dominio generan una respuesta 400 Bad Request, y cualquer otra excepción genera una respuesta 500 Internal Server Error.

Mediante el objeto IHostingEnvironment env (que se recibe mediante injección de dependencias) podemos validar si el microservicio se está ejecutando en un ambiente de desarrollo, para agregar más información en la respuesta. Si el ambiente es de pruebas o producción, ese contenido no se agregará.

```csharp
    public partial class HttpGlobalExceptionFilter : IExceptionFilter
    {
        public void OnException(ExceptionContext context)
        {
            logger.LogError(new EventId(context.Exception.HResult),
                context.Exception,
                context.Exception.Message);

            if (context.Exception.GetType() == typeof(BasketDomainException))
            {
                var json = new JsonErrorResponse
                {
                    Messages = new[] { context.Exception.Message }
                };

                context.Result = new BadRequestObjectResult(json);
                context.HttpContext.Response.StatusCode = (int)HttpStatusCode.BadRequest;
            }
            else
            {
                var json = new JsonErrorResponse
                {
                    Messages = new[] { "An error occurred. Try it again." }
                };

                if (env.IsDevelopment())
                {
                    json.DeveloperMessage = context.Exception;
                }

                context.Result = new InternalServerErrorObjectResult(json);
                context.HttpContext.Response.StatusCode = (int)HttpStatusCode.InternalServerError;
            }
            context.ExceptionHandled = true;
        }
    }
```

Por último, registramos el filtro en Startup.cs.

```csharp
    public IServiceProvider ConfigureServices(IServiceCollection services)
    {  
        services.AddMvc(options =>
            {
                options.Filters.Add(typeof(HttpGlobalExceptionFilter));

            });
    }
```

## Ejercicio

Cree la funcionalidad del ValidateModelStateFilter. Usando la propiedad context.ModelState, retorne un BadRequestObjectResult(error) que reciba como parámetro un objeto JsonErrorResponse con los errores de validación del modelo. Recuerde registrarlo en Startup.

Información extra sobre filtros:

* [Filters in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/mvc/controllers/filters?view=aspnetcore-2.2)
* [ASP.NET Core in Action - Filters](https://andrewlock.net/asp-net-core-in-action-filters/)
