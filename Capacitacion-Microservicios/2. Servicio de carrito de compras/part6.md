# Capacitación Microservicios: Módulo 2 - API Gateway

El API Gateway nos sirve para juntar las APIs de los microservicios que conforman el sistema, de modo que solo se tenga que hacer una petición desde el front.

La librería Ocelot nos permite configurar las redirecciones que tendrá el gateway.

La configuración del Gateway se guarda en un archivo con el nombre configuration.json, dentro del proyecto Web.Bff.Shopping.

La configuración de una redirección se separa en dos partes:

* La configuración upstream, que se refiere a lo que el gateway acepta como petición
* La configuración downstream, que se refiere a cómo y hacia dónde se hace la redirección

La configuración upstream tiene las siguientes propiedades:

* UpstreamPathTemplate: Define el formato de la petición entrante para esa redirección. Dentro de la ruta se pueden poner variables entre llaves, las cuales se pueden usar para construir la ruta de la redirección downstream.
* UpstreamHttpMethod: Define los métodos HTTP que aceptará la redirección con el formato especificado

La configuración downstream tiene las siguientes propiedades:

* DownstreamPathTemplate: Define el formato de la ruta hacia la cual se hace la redirección hacia el microservicio
* DownstreamScheme: Define el protocolo con el que se hace la redirección
* DownstreamHostAndPorts: Define a qué microservicio se hace la redirección. El host debe tener el mismo nombre que se le puso en el archivo docker-compose si se está ejecutando con Docker. Si no se usa docker, el host debe ser localhost con el puerto con el se ejecuta el microservicio.

```json
{
  "ReRoutes": [
    {
      "DownstreamPathTemplate": "/api/{version}/catalog/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "catalog.api",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/api/{version}/catalog/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST" ,"PUT", "DELETE" ]
    }
  ],
    "GlobalConfiguration": {
      "RequestIdKey": "OcRequestId",
      "AdministrationPath": "/administration"
    }
}
```

Si la redirección require autorización, se debe agregar las propiedades AuthenticationOptions abajo de UpstreamHttpMethod.

```json
{
  "ReRoutes": [
    {
        // Configuración de la ruta
        "AuthenticationOptions": {
            "AuthenticationProviderKey": "IdentityApiKey",
            "AllowedScopes": []
        }
    }
  ]
}
```

## Ejercicio

Agregue redirecciones para el servicio Basket.

* PUT y POST para "/api/{version}/basket" van hacia el servicio Basket
* GET y DELETE para "/api/{version}/basket/{everything}" van hacia el servicio Basket

## Extra

* Libro Microsoft, página 41