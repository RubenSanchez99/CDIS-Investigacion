# Capacitación Microservicios: Servicio de catálogo

## Modelo y EF Core

El primer paso es agregar las librerías de Entity Framework Core al proyecto del microservicio.


Entity Framework Core es un ORM que nos permite mapear las claes creadas en nuestro modelo a tablas de una base de datos relacional. EF Core se instala como un paquete NuGet.

EF Core nos permite elegir qué base de datos relacional usar. En los microservicios que hagan uso de una base de datos relacional, usaremos PostgreSQL.

Para instalar un paquete NuGet podemos usar los siguientes comandos dentro de la carpeta del proyecto.

```
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
```

O podemos usar la extensión NuGet Package Manager para instalar las dependencias mediante la paleta de comandos.

```
>NuGet Package Manager: Add Package
```

![](img/part4/nuget-add-package.png)

Procedemos a buscar el paquete que deseamos instalar en la ventana que aparece.

![](img/part4/nuget-search.png)

Seleccionamos el paquete de la lista de resultados.

![](img/part4/nuget-search-results.png)

Elegimos la versión estable más reciente.

![](img/part4/nuget-search-versions.png)

También es necesario agregar unas utilidades de línea de comandos que nos servirán al momento de crear migraciones del modelo de base de datos.

Abra el archivo .csproj y agregue el siguiente ItemGroup

```xml
<ItemGroup>
    <DotNetCliToolReference Include="Microsoft.VisualStudio.Web.CodeGeneration.Tools" Version="2.0.3" />
    <DotNetCliToolReference Include="Microsoft.EntityFrameworkCore.Tools.DotNet" Version="2.0.3" />
</ItemGroup>
```

Nuestro archivo .csproj debe quedar de la siguiente manera.

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>netcoreapp2.1</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <Folder Include="wwwroot\" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.App"/>
    <PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="2.1.2" />
  </ItemGroup>

  <ItemGroup>
    <DotNetCliToolReference Include="Microsoft.VisualStudio.Web.CodeGeneration.Tools" Version="2.0.3" />
    <DotNetCliToolReference Include="Microsoft.EntityFrameworkCore.Tools.DotNet" Version="2.0.3" />
  </ItemGroup>

</Project>
```

Después de agregar las librerías, aparecerá el siguiente mensaje en VS Code.

![](img/part4/vscode-restore.png)

Al presionar Restore, .NET va a descargar las liberías que hemos agregado al proyecto. Este paso es necesario cada vez que agregamos liberías nuevas. Este paso también se realizará automáticamente al momento de compilar, de ser necesario.

El siguiente paso es crear las clases que compondrán el modelo de este servicio.

Usaremos el flujo Code-First de Entity Framework Core, donde creamos nuestro modelo mediante clases de C#, y el framework genera tablas con las columnas correspondientes.

Cree una carpeta con el nombre "Model".

![](img/part4/new-folder.png)

Para crear una carpeta, haz clic derecho sobre la carpeta donde se desea crear la nueva carpeta y escoga la opción 'New Folder'.

Para agregar una clase, haga clic derecho sobre la carpeta donde desea crear el archivo y escoga la opción 'New C# Class' (La opción solo se encuentra disponible si se instaló la extensión 'C# Extensions').

![](img/part4/new-class.png)

![](img/part4/new-class-name.png)

Una clase de modelo de Entity Framework Core no requiere herencia ni implementar ninguna interfaz, solo necesita las propiedades que tendrá la entidad.

```csharp
namespace Catalog.API.Model
{
    public class CatalogBrand
    {
        public int Id { get; set; }

        public string Brand { get; set; }
    }
}
```

Podemos agregar una clase de configuración, que define cómo se construirá la tabla en la base de datos.

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Catalog.API.Model;

namespace Catalog.API.Infrastructure.EntityConfigurations
{
    class CatalogBrandEntityTypeConfiguration
        : IEntityTypeConfiguration<CatalogBrand>
    {
        public void Configure(EntityTypeBuilder<CatalogBrand> builder)
        {
            builder.ToTable("CatalogBrand");

            builder.HasKey(ci => ci.Id);

            builder.Property(cb => cb.Brand)
                .IsRequired()
                .HasMaxLength(100);
        }
    }
}
```

Para usar EF Core en nuestro proyecto, se debe crear una clase DbContext. Esta clase representa nuestra conexión con la base de datos.

```csharp
namespace Catalog.API.Infrastructure
{
    using Microsoft.EntityFrameworkCore;
    using EntityConfigurations;
    using Model;
    using Microsoft.EntityFrameworkCore.Design;

    public class CatalogContext : DbContext
    {
        public CatalogContext(DbContextOptions<CatalogContext> options) : base(options)
        {
        }

        // Agregamos un DbSet de la clase del modelo. Asignamos un nombre para la entidad.
        public DbSet<CatalogBrand> CatalogBrands { get; set; }

        protected override void OnModelCreating(ModelBuilder builder)
        {
            // Agregamos la clase de configuración de entidad
            builder.ApplyConfiguration(new CatalogBrandEntityTypeConfiguration());
        }
    }
}
```

Las configuraciones de las librerías que usamos en un proyecto de .NET Core se definen en la clase Startup. En el método ConfigureServices se configuran los servicios que proveen de la funcionalidad de las librerías agregadas, de modo que sean accesibles en las clases del proyecto (generalmente, mediante inyección de dependencias).

El método UseNpgsql es el que agrega una conexión con una base de datos de Postgres. Este método recibe como primer argumento el string de conexión para la base de datos. El segundo argumento es un método lambda donde se realiza configuración de la conexión con la base de datos.

```csharp
namespace Catalog.API
{
    public class Startup
    {
        public IServiceProvider ConfigureServices(IServiceCollection services)
        {
            services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_1);

            // Para EF Core
            services.AddEntityFrameworkNpgsql()
                .AddDbContext<CatalogContext>(options =>
            {
                options.UseNpgsql(Configuration["ConnectionString"],
                                     npgsqlOptionsAction: sqlOptions =>
                                     {
                                         sqlOptions.MigrationsAssembly(typeof(Startup).GetTypeInfo().Assembly.GetName().Name);
                                         //Configuring Connection Resiliency: https://docs.microsoft.com/en-us/ef/core/miscellaneous/connection-resiliency 
                                         sqlOptions.EnableRetryOnFailure(maxRetryCount: 5);
                                     });

                // Changing default behavior when client evaluation occurs to throw. 
                // Default in EF Core would be to log a warning when client evaluation is performed.
                options.ConfigureWarnings(warnings => warnings.Throw(RelationalEventId.QueryClientEvaluationWarning));
                //Check Client vs. Server evaluation: https://docs.microsoft.com/en-us/ef/core/querying/client-eval
            });
        }
    }
}
```

El objeto Configuration se recibe mediante inyección de dependencias en el constructor de la clase Startup.

```csharp
namespace Catalog.API
{
    public class Startup
    {
        public Startup(IConfiguration configuration)
        {
            Configuration = configuration;
        }

        public IConfiguration Configuration { get; }
    }
}
```

La configuración se carga de varias fuentes, que se obtienen en el siguiente orden (si se encuentra una configuración con el mismo nombre, se sobreescribe por la última fuente cargada).

* Archivo appsettings.json
* Archivo appsettings.{Environment}.json
* ASP.NET Core Secrets
* Variables de entorno
* Argumentos de línea de comando

```json
{
  "ConnectionString": "Server=127.0.0.1;Port=5432;Database=CapacitacionMicroservicios.CatalogDb;User Id=postgres;Password=pass;",
  "PicBaseUrl": "http://localhost:5101/api/v1/catalog/items/[0]/pic/",
  "Logging": {
    "IncludeScopes": false,
    "LogLevel": {
      "Default": "Debug",
      "System": "Information",
      "Microsoft": "Information"
    }
  },
  "AllowedHosts": "*"
}
```

Cuando hagamos validaciones de reglas de negocio, generalmente definimos una clase de exepción que se lanza solamente cuando se intente romper una de estas reglas. A este tipo de exepción se le llama exepción de dominio, y en este microservicio su clase correspondiene tendrá el nombre de CatalogDomainException.

```csharp
using System;

namespace Catalog.API.Infrastucture.Exceptions
{
    public class CatalogDomainException : Exception
    {
        public CatalogDomainException()
        { }

        public CatalogDomainException(string message)
            : base(message)
        { }

        public CatalogDomainException(string message, Exception innerException)
            : base(message, innerException)
        { }
    }
}
```

## Ejercicio

Agregue el Entity Framework Core al proyecto y cree las siguientes tres clases en el modelo:

* CatalogType
  * Id (int)
  * Name (string)
* CatalogBrand
  * Id (int)
  * Name (string)
* CatalogItem
  * Id (int)
  * Name (string)
  * Description (string)
  * Price (decimal)
  * AvailableStock (int)
  * CatalogTypeId
  * CatalogType
  * CatalogBrandId
  * CatalogBrand
  * RemoveStock (Recibe una cantidad y la reduce de AvailableStock. Lanza CatalogDomainException si la cantidad a restar es menor o igual a cero)
  * AddStock (Recibe una cantidad y la agrega a AvailableStock)

Cree clases de configuración de entidades para las tres clases del modelo:

* CatalogType
  * Id: Autogenerar
  * Name: Marcar como requerido y limitar a 100 caracteres
* CatalogBrand
  * Id: Autogenerar
  * Name: Marcar como requerido y limitar a 100 caracteres
* CatalogItem
* Id (int)
  * Cambiar nombre de tabla a 'Catalog'
  * Id: Autogenerar
  * Name: Marcar como requerido y limitar a 50 caracteres
  * Price: Marcar como requerido
  * CatalogTypeId: Configurar como relación uno-a-muchos y agregar foreign key
  * CatalogBrandId: Configurar como relación uno-a-muchos y agregar foreign key

## Material extra

 * https://ardalis.com/how-to-add-a-nuget-package-using-dotnet-add
 * https://docs.microsoft.com/en-us/ef/core/miscellaneous/cli/dotnet
 * https://github.com/aspnet/EntityFramework.Docs/blob/master/entity-framework/core/miscellaneous/configuring-dbcontext.md
 * http://www.entityframeworktutorial.net/efcore/entity-framework-core.aspx
 * https://docs.microsoft.com/en-us/ef/core/get-started/aspnetcore/new-db