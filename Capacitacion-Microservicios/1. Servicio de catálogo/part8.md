# Capacitación Microservicios: Servicio de catálogo

## Prueba con Postman

Para probar la API de nuestro servicio, usaremos la aplicación Postman.

Postman nos permite enviar peticiones de HTTP a las rutas expuestas por la API.

Para crear un producto en el catálogo, se envía una petición POST a la ruta /api/v1/catalog/items, definida en CatalogController.cs
```
POST http://localhost:5101/api/v1/catalog/items
```

El método CreateProduct recibe como parámetro un CatalogItem con el atributo [FromBody]. Esto significa que el cuerpo de la petición debe contener los datos de un CatalogItem en formato JSON.

Configuramos el ContentType y el cuerpo de la petición en Postman con la siguiente información.

```json
{
	"Name": "Batman Shirt",
	"Price": "6.24",
	"CatalogTypeId": "0",
	"CatalogBrandId": "0"
}
```

Recuerde seleccionar el formato 'JSON (application/json)' en el menú a la derecha de la opción 'binary'. No especificar esto puede provocar un error que retorna una respuesta 515.

![](img/part8/postman-post.png)

Al presionar Send, Postman envía la petición y debe retornar una respuesta 201 Created.

![](img/part8/postman-post-response.png)

Confirmamos que el registro está en la tabla.

```sql
SELECT Id, Name, Description, Price, PictureFileName, CatalogTypeId, CatalogBrandId, AvailableStock, RestockThreshold, MaxStockThreshold, OnReorder
FROM [CapacitacionMicroservicios.CatalogDb].dbo.[Catalog];
```

Para revisar la consulta desde la API, envíe una petición GET a la siguiente ruta.

![](img/part8/postman-get.png)

![](img/part8/postman-get-response.png)

Podemos comitear los cambios desde VS Code.

Primero agregamos los archivos que cambiamos. Seleccione la pestaña Git en la barra del lado izquierdo (la que tiene el ícono con nodos). Ahí verá una lista con todos los archivos modificados o no registrados.

Presione el botón con el tooltip 'Stage all changes', o agrege individualmente los archivos modificados.

![](img/part8/git-stage.png)

Usando la paleta de comandos, ejecute el siguiente comando.

![](img/part8/git-commit.png)

Para terminar un feature, usamos el siguiente comando en la paleta de comandos.

```
>gitflow feature finish
```

Esto hará el merge con la rama de develop y cerrará la rama del feature.