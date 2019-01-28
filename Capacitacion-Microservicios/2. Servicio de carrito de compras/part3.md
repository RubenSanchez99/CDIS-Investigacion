# Capacitación Microservicios: Módulo 2 - Validaciones del modelo

Podemos agregar validaciones a clases del model de modo que podamos validar que los datos que reciban los ActionMethods sean correctos.

Para realizar estas validaciones, podemos usar DataAnnotations de ASP.NET Core, o podemos usar FluentValidation.

Para hacer validaciones con [FluentValidation](https://fluentvalidation.net/), se crea una clase que herede de AbstractValidator\<T>, donde T es la clase del modelo sobre la cual se va a validar.

Dentro del constuctor se registran las reglas para validar la clase del modelo. Se puede usar el método WithMessage para que la respuesta de la petición retorne el mensaje de error.

```csharp
namespace Basket.API.Validators
{
    public class CustomerBasketValidator : AbstractValidator<CustomerBasket>
    {
        public CustomerBasketValidator()
        {
            RuleForEach(x => x.Items)
                .Must(x => x.Quantity > 0)
                .WithMessage("Invalid number of units");
        }
    }
}
```

Se registra FluentValidation después de registrar MVC y se registra el validador que agregamos.

```csharp
        public IServiceProvider ConfigureServices(IServiceCollection services)
        {
            // Add framework services.
            services.AddMvc(options =>
            {
                options.Filters.Add(typeof(HttpGlobalExceptionFilter));

            }).AddControllersAsServices()
                .AddFluentValidation();

            services.AddTransient<IValidator<CustomerBasket>, CustomerBasketValidator>();
        }
```

## Ejercicio

Agregue un validator que valide la cantidad de los items de CustomerBasket. El controlador debe retornar BadRequest si se intenta agregar un basket con objetos con cantidades menores a 1.