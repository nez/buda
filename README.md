
#BUDA

[![Stories in Ready](https://badge.waffle.io/mxabierto/buda.png?label=ready&title=Ready)](https://waffle.io/mxabierto/buda)
-----

Bus de Datos Abiertos

### API

BUDA está disponible en `api.datos.gob.mx`

El catálogo de datasets se puede consultar en `http://api.datos.gob.mx/v1/catalog.json`

Por conveniencia, se recomienda instalar `httpie` para interactuar con BUDA, los siguientes ejemplos están escritos contemplando que tienes httpie.

Consultar datasets registrados:
`http GET api.datos.gob.mx:32771/`

Registrar un nuevo dataset en el manager:
`http POST api.datos.gob.mx:32771/ format=yaml dataset=@DATASET.bdds`

Borrar un dataset:
`http DELETE api.datos.gob.mx:32771/a0dcf7b487d319d682c6cb670fb23c11e593d7155dbb9578ccab9ad6fbd77166`

Cargar datos:
`cat ARCHIVO_CON_DATOS | nc api.datos.gob.mx PORT`

Y finalmente queda publicado y se puede consultar desde el front:
`http://api.datos.gob.mx/v1/DATASET`

Este es un ejemplo de como hacer consultas:
`http://api.datos.gob.mx/v1/data-fusion?catalog-dataset-resource.mediaType=Excel`

### Arquitectura

__Master:__ buda

Ejecutable binario que manipula proceso(s) manager; se comunica con el manager
mediante su interfaz tipo REST

__Manager:__ buda-manager

Proceso que se ejecuta en modo daemon y se encarga de administrar un catálogo de datos
formado por distintos datasets; conserva en todo momento una lista de los datasets activos
y sus respectivos agentes

__Datasets:__ csv_sample.bdds

Entradas independientes en el catálogo de datos; se describen mediante un archivo
de especifición ( JSON o YAML ); estos archivos son utilizados por el manager para
crear agentes; un agente es asignado a cada dataset; a cada dataset es asignado
un ID que se calcula como un SHA a partir del archivo utilizado para crearlo

__Agents:__ buda-agent-{data.type}

Proceso tipo de worker, se inicia por el manager y ejecuta en modo daemon;
abre un stream de lectura en el "hotspot" marcado por su dataset y espera por datos
de entrada; cada paquete de datos recibido es procesado y almacenado de
acuerdo a las configuraciones de la zona; los agentes son procesos especializados
de acuerdo al tipo de dato que se procesará en el dataset; cada dataset indica
tambien donde debera reportar errores el agente mediante un stream de escritura

#### Agentes disponibles

agent/csv

agent/geojson

agent/json

agent/jsonl

agent/xml

__Front:__ buda-front

Ademas del manager, front es el otro proceso de primer nivel en la plataforma; se
encarga de presentar una interfaz de consumo mediante HTTP/S al usuario para realizar
consultas de toda la información procesada y almancenada en los distintas datasets que
conformán el catálogo

### FAQ

__Qué se pretende?__

Definir un diseño modular para la plataforma que cumpla con los requerimientos de la misma basado en la filosofía de UNIX:
> Pequeños programas que hacen una cosa y la hacen bien. [+](http://en.wikipedia.org/wiki/Unix_philosophy#Program_Design_in_the_UNIX_Environment)

__Diseño != Implementación__

Prioritariamente se debe trabajar e invertir tiempo en la definición, refinación y justificación del diseño; pensado desde el punto de vista de la plataforma y agnóstico a algúna tecnología especifica; debe ser lo más sencillo y claro posible.

__Cuales son las metas principales a alcanzar?__

Idealmente la plataforma debe:

- Poder procesar de forma independiente datos en distintos formatos y de distintas fuentes
- Ser fácilmente extensible en el futuro para soportar distintos formatos y fuentes de datos
- Realizar el procesamiento de forma eficiente en terminos de consumo de recursos ( CPU, memoria, etc )
para poder trabajar con grandes cargas de datos
- Deberá soportar [streams](http://goo.gl/Tp9Dm) de datos para eficientar el manejo de grandes cantidades de información
- Presentar un API tipo REST para poder consultar de forma flexible y eficiente los datos procesados y almacenados
- Permitir el desarrollo de herramientas visuales para tareas de administración y consumo de información
- Ser agnóstica en cuanto a tecnología de forma que sus componentes puedan ser escritos incluso en distintos lenguajes de programación favoreciendo así un diseño [loosely coupled](http://en.wikipedia.org/wiki/Loose_coupling)

### Consideraciones

- Cada componente de la arquitectura es independiente, el mantenimiento del proyecto
  es sencillo y se puede realizar en paralelo en distintos frentes
- Para agregar soporte a distintos tipos de datos ( XML, feeds, etc ) basta con escribir
  un agente adecuado por lo que la plataforma es extensible
- Implementar esta arquitectura como un ambiente de microservicios es simple porque todas
  las capas estan bien delimitadas, son sencillas e independientes entre sí
- El eje central de la arquitectura es la especificación de dataset y esto puede desarrollarse
  y mejorarse con el apoyo de la comunidad :)
- Todos los componentes son procesos livianos y utilizan streams de datos por lo que los
  requerimientos de CPU y memoria son bajos
- Un _master_ puede fácilmente comunicarse con distintos _managers_ e iniciar distintos
  _fronts_ por lo que el sistema puede manejarse de forma distribuida
- Los agentes son solo procesos hijos asi que la implementación específica es un detalle
  irrelevante para el resto de la plataforma; i.e. pueden escribirse agentes en Python,
  Ruby, Go, etc
- En una versión para producción los agentes incluso pueden desplegarse como containers
  en el cluster de forma relativamente sencilla
- Los datasets son solo un archivo JSON/YAML por lo que implementar herramientas visuales para
  diseñarlos es trivial
- Ya que los datasets son archivos de texto plano se puede montar un sistema de entrega continua
  de forma sencilla, por ejemplo: se almacenan en un repositorio Git, se configuran
  hooks de 'pre' y 'post' update que mediante al manager 'eliminan' y/o '(re)inician' los datasets
  que han sido modificados
- El _manager_ implementa una interfaz RESTful HTTP para comunicarse con el master por lo que
  implementar herramientas visuales para administrarlos es trivial
- Los datasets incluyen un elemento _hotspot_ que no es mas que el endpoint del stream
  utilizado para recibir datos; estos endpoints pueden ser UNIX sockets, TCP sockets o
  algun otro mecanismo más avanzando ( por ejemplo torrents para implementarse como sistemas
  realmente descentralizados :sunglasses: )
- En caso de utilizar UNIX sockets se pueden "montar" en la maquina local de forma segura
  utilizando un SSH tunnel; en caso de sockets TCP se pueden implementar de forma segura
  mediante conexiones HTTPS
- Una vez creado el dataset se pueden implementar herramientas visuales tipo drag & drop para
  enviar datos de forma sencilla ya que lo único que la herramienta debe hacer es iniciar
  el streaming de paquetes hacia el hotspot utilizado
