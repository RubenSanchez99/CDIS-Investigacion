# Capacitación Microservicios: Módulo 1 - Migraciones

EF Core posee un sistema de migraciones. La utilidad de esto es que podemos llevar un control de los cambios hechos al modelo. También podemos actualizar la base de datos al nuevo modelo o restaurarla a un punto anterior.

Cada que se haga un cambio en la base de datos, se debe ejecutar el siguiente comando dentro de la carpeta del proyecto. Después del 'add' se pone un mensaje corto describiendo el cambio hecho en la base de datos.

La etiqueta Context especifica el DbContext sobre el cual se hará la migración. La etiqueta -o especifica la carpeta donde se pondrán los archivos de migración.

```bash
dotnet ef migrations add Initial-Migration --context Catalog.API.Infrastucture.CatalogContext -o Infrastructure/CatalogMigrations
```

Esto genera tres archivos en la carpeta CatalogMigrations. Los archivos que tienen el nombre Initial-Migration definen qué tablas y cambios se agregan a la base de datos, junto con el proceso para removerlos. El archivo ModelSnapshot define un modelBulder que genera las tablas de la última versión.

Es recomendable que se agreguen cambios pequeños a las migraciones, para evitar migraciones demasiado grandes. Haga una migración cada que agregue datos al modelo o agregue clases nuevas.

Usaremos una imagen de Docker para levantar la base de datos. Para levantarla, ejecute el siguiente comando en la carpeta donde se ubica el archivo docker-compose.yml.

```bash
docker-compose -f docker-compose.infrastructure.yml up sql.data
```

Esto levantará el servidor de base de datos de Postgres. Podemos usar pgAdmin 4 para visualizarla en un navegador.

Para actualizar la base de datos a la última versión, ejecute el siguiente comando en la terminal.

```bash
dotnet ef database update
```

## Ejercicio

Crear base de datos de Catalog mediante migraciones.