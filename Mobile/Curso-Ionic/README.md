# Curso Ionic

## Prerequisitos

1. Instalar Node.js para tener acceso a npm -> <https://nodejs.org>
2. Instalar Angular CLI mediante el comando -> ```npm install -g @angular/cli```
3. Instalar Ionic siguiendo las instrucciones -> <https://ionicframework.com/getting-started#cli>
4. Instalar SDK de la plataforma a publicar -> <https://developer.android.com/studio>
5. Instalar un editor de texto -> <https://code.visualstudio.com/>
6. Opcionalmente, instalar extensiones de Visual Studio Code
    * TSLint
    * Prettier - Code formatter
    * Debugger for Chrome
    * npm
    * npm Intellisense
    * Auto Import
    * Cordova Tools
    * Ionic 4 Snippets
    * Auto Close Tag
    * Auto Rename Tag
    * HTML Snippets
    * Angular Schematics
    * Angular v7 Snippets
    * Angular 2, 4 and upcoming latest TypeScript HTML Snippets

## Índice

### Introducción

* Introducción a Ionic 4
* Introducción a Cordova

### Creación de pantallas

* Uso de Ionic CLI para creación de proyecto
* Estructura de proyecto de Ionic 4
* Creación de pantallas con Ionic CLI
* Angular Router y navegación en Ionic
* Componentes para mostrar listas
  * List
  * Refresher
  * Toolbar
* Consultas a API mediante HttpClient
* Componentes para formularios
  * Input
  * Date and Time Pickers
  * Select
  * Button
  * Toast
* Envío de datos mediante HttpClient

### Seguridad

* Funcionamiento de servicio de identidad de back-end
  * JSON Web Tokens
  * Hybrid Grant Type
* Funcionamiento de SecurityService
* Almacenando token de seguridad con LocalStorage
* Uso de HttpClient con Bearer token

### Ejecución en dispositivos

* Ejecución en emulador de Android
* Ejecución en dispositivo físico de Android
* Ejecución en dispositivo físico de iOS

### Plugins de Cordova

* Plugin de cámara
* Plugin de Google Maps
* Plugin de transferencia de archivos

## Actividades

### Creación de proyecto ionic

### Configuración de dispositivo/emulador

### Pantalla de consulta de productos

Muestra un listado de productos obtenidos de una API

* Componentes
  * ion-list y ion-item
  * Refresher (pull to refresh)

### Login

Solicita al usuario un usuario y una contraseña para entrar a la aplicación

* Componentes
  * JSON Web Tokens
  * Local Storage

### Pantalla de alta de productos

Permite al usuario agregar un producto al catalogo

* Componentes
  * Modal
  * cordova-plugin-camera
  * cordova-plugin-file-transfer
  * Forms
  * Toast

### Pantalla de sucursales

Muestra un mapa con sucursales de la tienda

* Componentes
  * Menú lateral
  * Angular Router
  * cordova-plugin-googlemaps