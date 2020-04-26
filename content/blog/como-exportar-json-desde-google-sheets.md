---
title: "Cómo exportar JSON desde Google Sheets"
date: 2020-04-25T08:54:53-05:00
draft: false
author: "David Daza"
description: "Utiliza Google Apps Script para exportar los registros de una base de datos disponible en Google Sheets en formato JSON."
---

Respuesta rápida: utilizando Google Apps Script.

## ¿Qué es Google Apps Script?

[Google Apps Script](https://developers.google.com/apps-script/) (GAS) es una plataforma para el desarrollo de aplicaciones web que se conectan con las librerías de G Suite. Lo único que hay que hacer es escribir Javascript. No hay nada que instalar. GAS provee un editor de código online que se ejecuta en los servidores de Google.

### Tipos de script

Hay tres tipos: [standalone](https://developers.google.com/apps-script/guides/standalone), [container-bound](https://developers.google.com/apps-script/guides/bound) y [web apps](https://developers.google.com/apps-script/guides/web).

Para este caso, los scripts de tipo web apps son perfectos porque permiten exponer en la red cierta funcionalidad. El único requisito que debe cumplir un script de este tipo es tener una función **`doGet`** o **`doPost`** que al ser ejecutada devuelva un objeto de tipo **`HtmlOutput`** o **`TextOutput`**.

De hecho, cualquier script standalone o container-bound puede ser migrado a un script web app si cumple los criterios mencionados anteriormente.

## Manos a la obra

Para este ejemplo he creado una base de datos que contiene las [coordenadas geográficas de las capitales de Sudámerica](https://docs.google.com/spreadsheets/d/1gPGgRxrnkgUwc6HKB88DiCeEXb6MQBc-1JLN_3ag4ss/edit#gid=0). Es necesario aclarar que se **debe ser el propietario** del spreadsheet sobre el cual se va a ejecutar el script.

Con la base de datos lista es momento de crear el script desde el [panel de control de Google Apps Script](https://script.google.com/home/start) haciendo clic en el botón que dice _Nuevo proyecto_. La acción anterior lleva a un editor de texto donde es posible escribir código Javascript.

El primer paso es leer los valores que existen en la base de datos:

```js
function readSpreadsheet () {
  // Abrir un spreadsheet por su id
  const spreadsheet = SpreadsheetApp.openById(
    '1gPGgRxrnkgUwc6HKB88DiCeEXb6MQBc-1JLN_3ag4ss'
  );

  // Seleccionar la hoja que tiene los datos
  const sheet = spreadsheet.getSheets()[0];

  // Obtener los valores que tiene
  const values = sheet.getDataRange().getValues();
}
```

La clase [SpreadsheetApp](https://developers.google.com/apps-script/reference/spreadsheet/spreadsheet-app) define la API que permite manipular Google Sheets. El método **`openById`**, en la línea 3, hace exactamente lo que traduce. Devuelve una [hoja de cálculo](https://developers.google.com/apps-script/reference/spreadsheet/spreadsheet) en base a su _id_.

El id es la porción de texto entre `/d/` y `/edit` de la url de la base de datos. Por ejemplo, para la url *https://docs.google.com/spreadsheets/d/1gPGgRxrnkgUwc6HKB88DiCeEXb6MQBc-1JLN_3ag4ss/edit#gid=0* el id es `1gPGgRxrnkgUwc6HKB88DiCeEXb6MQBc-1JLN_3ag4ss`

El método **`getSheets`**, en la línea 8, devuelve un arreglo con todas las hojas (tablas) que existen dentro de la base de datos. El índice 0 selecciona la primera hoja, pues es donde están guardados los datos.

En la línea 11, las funciones **`getDataRange`** y **`getValues`** son las responsables de obtener todos los registros. En este momento, si se hiciera un *Logger.log* (el console.log de GAS) de `values` el resultado sería el siguiente:

```js {linenos=false}
[
  ['pais', 'capital', 'latitud', 'longitud'],
  ['Argentina', 'Buenos Aires', '-34.599722', '-58.381944'],
  ['Bolivia', 'La Paz', '-16.494167', '-68.1475'],
  [...],
]
```

Es un arreglo bidimensional. Cada arreglo interno representa una fila de base de datos. Ahora el objetivo es pasar de ese formato a un objeto que pueda ser parseado como JSON.

```js {linenostart=13}
// Obtener los headers de la base de datos
const headers = values.shift();

// Inicializar objeto de respuesta
const response = { data: [] };

// Agregar datos
for (let i = 0; i < values.length; i++) {
  let row = values[i];
  let register = {};
  for (let j = 0; j < row.length; j++) {
    register[headers[j]] = row[j];
  }
  response.data.push(register);
}

// Parsear JSON
const json = JSON.stringify(response);
```
El código anterior es el encargado de pasar del formato de arreglo bidimesional a un objeto JSON.

```json {linenos=false}
{
  "data": [
    {
      "pais": "Argentina",
      "capital": "Buenos Aires",
      "latitud": -34.599722,
      "longitud": -58.381944
    },
    { ... },
    { ... },
  ]
}
```

Por supuesto, existen diversas maneras de lograr el mismo objetivo, pero es mejor mantener el código simple por el momento.

Ahora bien, el script aún no cumple con los requisitos mencionados anteriormente para ser tipo web app. Hora de refrescar la memoria:

1. El script debe exponer una función `doGet` o `doPost`
1. Cuando cualquiera de las funciones se ejecute debe retornar un objeto de tipo [`HtmlOutput`](https://developers.google.com/apps-script/reference/html/html-output?hl=en) o [`TextOuput`](https://developers.google.com/apps-script/reference/content/text-output)

El primer requisito es sencillo. Renombrar la función `readSpreadsheet` a `doGet` lo satisface. ¿Por qué doGet y no doPost? Muy simple, doGet se ejecuta en peticiones HTTP de tipo GET que se hagan al script.

El segundo es un poco más complejo, pero nada que no pueda ser resuelto después de revisada la documentación.

```js {linenostart = 32}
return ContentService.createTextOutput(json).setMimeType(
  ContentService.MimeType.JSON
);
```
El código ya está listo.

```js
function doGet() {
  // Abrir un spreadsheet por su id
  const spreadsheet = SpreadsheetApp.openById(
    '1gPGgRxrnkgUwc6HKB88DiCeEXb6MQBc-1JLN_3ag4ss'
  );

  // Seleccionar la hoja que tiene los datos
  const sheet = spreadsheet.getSheets()[0];

  // Obtener los valores que tiene
  const values = sheet.getDataRange().getValues();

  // Obtener los headers de la base de datos
  const headers = values.shift();

  // Inicializar objeto de respuesta
  const response = { data: [] };

  // Agregar datos
  for (let i = 0; i < values.length; i++) {
    let row = values[i];
    let register = {};
    for (let j = 0; j < row.length; j++) {
      register[headers[j]] = row[j];
    }
    response.data.push(register);
  }

  // Parsear JSON
  const json = JSON.stringify(response);

  return ContentService.createTextOutput(json).setMimeType(
    ContentService.MimeType.JSON
  );
}
```
El siguiente paso es publicar el script como una aplicación web seleccionando la opción *Implementar como aplicación web* desde el menú *Publicar* presente en la barra de herramientas.

{{< figure src="/media/gas_publish.webp" caption="Implementar script como aplicación web" >}}

En el cuadro de diálogo que se presenta es necesario brindar acceso a cualquier persona. Esto no quiere decir que la base de datos sea pública. Quienes ejecuten el script solo tienen permisos de lectura. De esta manera, la base de datos puede tener decenas de tablas diferentes, pero solo acceso a las definidas en el script.

{{< figure src="/media/gas_dialog.webp" caption="Permisos de ejecución" >}}

Luego de desplegar la aplicación, la plataforma ahora presenta un cuadro de diálogo con la URL encargada de ejecutar el script.

{{< figure src="/media/gas_result.webp" caption="URL para ejecutar script" >}}

Para comprobar el correcto funcionamiento del script basta con copiar y pegar la URL en la barra de direcciones del navegador.

Ya se puede acceder a los datos desde una aplicación web, por ejemplo. La mejor parte es que cada vez que se actualicen o agreguen valores a la hoja de cálculo se actualizará, también, la información devuelta por el script.

Esto es solo la punta del iceberg y las posibilidades son muchas. El script podría leer todas las hojas que existen en la base de datos o devolver los datos en formato CSV y no en formato JSON. Incluso, un script podría recibir como parámetro el id de la hoja de cálculo que se quiere leer y tener una especie de API propia.

También, es clave recordar que Google Apps Script tiene una API para todos los servicios de G Suite, no solo para Google Sheets, por tanto, lo que se puede lograr con esta plataforma sólo está limitado por la imaginación de cada uno.
