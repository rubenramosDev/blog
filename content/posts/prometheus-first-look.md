---
title: "Monitereo de servicios con Prometheus | 1"
languageAltTitle: "First look at Prometheus"
hideFooter: true
date: 2022-07-30T11:30:04+03:00
draft: false
url: "/post/prometheus-basics-1"
summary: "Introducción, conceptos clave y explicación del funcionamiento de esta herramienta para el monitoreo de servicios."
tags: ["métricas", "prometheus", "microservicios", "monitoreo"]
categories: ["métricas", "prometheus", "microservicios", "monitoreo"]
author: "Ruben Ramos"
images: "images/post-1/cover.jpg"
keywords: ["métricas", "prometheus", "microservicios", "monitoreo"]
cover:
    image: images/post-1/cover.jpg
    alt: "primer vistazo a Prometheus."
---

## Introducción al monitoreo de aplicaciones, ¿Para qué tomarse la molestia?
En el mundo de los microservicios *(y en general, en cualquier arquitectura)* es de suma importancia conocer el estado de los servicios en funcionamiento.

Cuando menciono estado, me refiero a la capacidad de responder a preguntas del tipo ¿Cuántas peticiones por minuto se reciben?, ¿Cuál es el tiempo de respuesta promedio?, ¿Cuál es el porcentaje de errores por minuto?, ¿Cuál es el endpoint más solicitado?, ¿Cuál es el número de peticiones activas?.

A la supervisión continua de un conjunto de métricas de rendimiento *(o de cualquier tipo)*, se le conoce como **monitoreo de aplicaciones**.  

Si bien, el simple hecho de monitorear nuestras aplicaciones no evita que estás puedan colapsar o que los tiempos de respuesta se degraden en ciertas horas del día, **sí nos brinda la capacidad de conocer con exactitud cuál es su estado** y eventualmente, la capacidad de **gestionar nuestro conjunto de servicios** mediante la toma de acciones preventivas o correctivas, como puede ser el despliegue de servidores extras en ciertas horas del día o la implementación de un balanceador de carga por monitoreo, etc. 

Todo esto con un objetivo único, **garantizar un funcionamiento óptimo e ininterrumpido de nuestros servicios y, naturalmente, una buena experiencia para el usuario final.**

### Beneficios del monitoreo
Google ha realizado numerosos esfuerzos para propagar la cultura del monitoreo, existen innumerables fuentes de información y documentación generada por ellos mismos, en donde comparten sus experiencias en el área y sus recomendaciones.

En esta sección voy a retomar el apartado Why Monitor? del artículo Monitoring Distributed Systems [1]. Extiendo mi amplia recomendación para la consulta de este artículo que es, sin duda, muy enriquecedor.

Me centraré en 4 puntos claves que considero los más redituables del monitoreo de nuestros servicios:
- **Análisis de tendencias a largo plazo**
  - Tener un histórico de un valor nos permite realizar proyecciones del mismo. Por ejemplo, ¿Cuál es la tendencia de ventas de cierto artículo? o ¿Cuál es la proyección de ventas?
- **Realizar comparaciones en diferentes períodos de tiempo**
  -  ¿Cuál es la cantidad de peticiones recibidas en el mismo período del año pasado? ¿Cuál es la comparativa de ventas en diferentes épocas del año?
- **Análisis retrospectivo de sucesos**
  - Supongamos que esperabamos realizar un cierto número de ventas en un fin de semana clave para nuestro negocio pero el objetivo quedó muy por debajo del esperado. ¿Qué sucedió? ¿Nuestros servicios fueron factor? Bueno, gracias al monitereo que estamos realizando podemos analizar el rendimiento de nuestros servicios, saber si estos estuvieron recibiendo un tráfico excesivo que terminó por afectar su rendimiento, otro escenario sería si nuestros proveedores externos experimentaron fallas en su sistema que terminó por afectar a la experiencia de usuario de nuestros clientes y eso, a su vez, al número de ventas.
- **Alertas**
  - Retomando el ejemplo anterior, ¿Es necesario esperar a que nuestros sistemas colapsen para poder hacer algo al respecto? **La respuesta es un rotundo no.** Un gran poder que nos brinda el monitereo es el tema de las Alertas. Gracias al monitoreo, seremos capaces de establecer reglas que nos permitan identificar tendencias dentro de nuestro sistema para poder actuar con antelación. Por ejemplo, si el tiempo de respuesta comienza a degradarse, una alerta llegará a los encargados para que estos puedan desplegar un servicio extra y aliviar el estrés.   
  En un segundo artículo de esta serie, estaré abordando el tema de **Alertmanager** que es una herramienta de Prometheus cuyo propósito es el de controlar y gestionar las alertas que fijemos en nuestro sistema.

## Prometheus, el rey del monitoreo.
Prometheus es un **sistema de monitoreo de código abierto** cuyo principal función se centra en  **recopilar, almacenar y por último, exponer** la información de un conjunto de servicios. 

Si bien, esta serie de artículos se centrará en el monitoreo de microservicios, Prometheus puede monitorear **cualquier otro servicio capaz de exponer sus métricas**, siempre y cuando esta información cuente con un **formato específico**. Un ejemplo de otros servicios, en donde es posible y sería interesante el monitoreo, es en los servidores Linux, Windows, NGINX o servicios como Kafka o bases de datos como MySQL, SQL Server, etc.

La siguiente imagen es un ejemplo del tipo de resultados que se pueden obtener de la mano de Prometheus y Grafana. En donde Prometheus es el encargado de consolidar la información de los servicios y Grafana es el encargado de la parte visual, la generación de gráficos y dashboards.
![ejemplo-métricas](https://miro.medium.com/max/1312/1*dzODOIDqXTsWauf5eM8Trw.gif#center)

## Entendiendo su arquitectura y funcionamiento
La siguiente imagen muestra la arquitectura de Prometheus de forma muy general, no obstante, nos ayuda a comprender su funcionamiento interno. 

Como se mencionó anteriormente, Prometheus nos ayuda a **recopilar, almacenar y exponer** las métricas obtenidas de nuestros servicios. Para entender cómo se lleva a cabo este proceso analicemos la siguiente imagen:
![arquitectura-básica-Prometheus](https://rubenramosdev.github.io/blog/images/post-1/promethus-architecture.jpg#center)

1. **Retrival**: Es un motor de *scrapping* encargado de **recolectar** la información de los diferentes servicios *(conocidos como Jobs)* que nos interesa monitorear. En un intervalo de tiempo definido por nosotros, por ejemplo, cada 20 segundos, 2 o 5 minutos, este motor se encargará de consultar los servicios registrados para recolectar su información. Los intervalos de tiempo pueden ser generales para todos los *Jobs* o cada *Job* puede tener un tiempo particular.

2. **Time series database o TSDB**: Una vez que la información ha sido recolectada, Prometheus **almacena** la misma en una base de datos de tipo **Time series** *(se explica más adelante)*. De forma nativa, Prometheus ya cuenta con la integración de una base de datos para almacenar la información de forma local, en disco. No obstante, se pueden integrar alternativas para almacenar la información de forma remota.

3. **HTTP Server**: Prometheus cuenta con un servidor HTTP integrado. Este servidor **expone** la información previamente recolectada y almacenada por medio de una ruta http, */metrics*. Con la información disponible para ser consultada de esta manera, se pueden utilizar múltiples aplicaciones especializadas para la creación de gráficos, como puede ser Grafana.

## Métricas, ¿Qué son?
En el sitio oficial de Prometheus, se presenta la siguiente definición:
> In layperson terms, metrics are numeric measurements. Time series means that changes are recorded over time. 

Sin más, **una métrica es una medición numérica.**

No obstante la definición va más allá de eso y nos comienza a introducir un concepto clave de Prometheus y es **Time series**, en español **Serie Temporal**. De manera general, una serie temporal es una sucesión de datos que han sido recabados en diferentes momentos y ordenados cronológicamente.

La siguiente imagen nos puede ayudar a entender de mejor manera el concepto, se trata de una animación del tipo de gráfica que se puede lograr con una serie temporal:
![gráfica-con-datos-series-temporal](https://media.giphy.com/media/rM0wxzvwsv5g4/giphy.gif#center)

Gracias a la forma en que se recolecta y organiza la información, se puede lograr ver las tendencias o comportamiento de ciertos valores, por ejemplo, el porcentaje de ventas en cierto período del año o el nivel de carga de nuestros servicios en diferentes horas del día, etc. 

### Formato de las métricas, ¿Cómo esta compuesta una métrica?

Ahora que ya tenemos un contexto más amplio acerca de Prometheus, es momento de entrar en detalles más técnicos. Como fue mencionado anteriormente, Prometheus puede recolectar la información de múltiples servicios, poco importa si se trata de un microservicio o un servidor Linux, la única condición es que esta información tenga un formato específico para que pueda ser interpretada, recolectada y almacenada eventualmente.

En la documentación oficial de Prometheus [2], se brinda el siguiente ejemplo para demostrar la composición de una métrica:

```
<nombre de la métrica>{<nombre etiqueta>=<valor de la etiqueta>, ...}
```

A continuación, vamos a analizar una métrica real que se obtuvo de un microservicio. En este caso, el objetivo de la métrica es dar a conocer el número de peticiones que se encuentran activas en el servidor:
```
application_httprequests_active{server="SERVIDOR-EJEMPLO"} 3
```

- **application_httprequests_active**: Nombre de la métrica.
- **server=SERVIDOR-EJEMPLO**: Este valor es conocido como una **etiqueta o label**. Una etiqueta es información complementaria para precisar o identificar de mejor manera el origen de la información, adelante veremos más ejemplos.  
- **3** : Valor de la métrica.

Con esta métrica, podemos entender que **en el servidor con nombre "SERVIDOR-EJEMPLO" se encuentran 3 peticiones activas en el momento en que fue consultado.**

## Tipos de métricas
Prometheus implementa 4 tipos de métricas diferentes para almacenar su información:

### Counter
Counter representa el tipo de medición cuyo valor es solamente acumulativo, es decir, **incrementa y no disminuye**. Ejemplos: *Número de peticiones a un servicio, núm. de errores o las ventas de algún producto.*
```
application_httprequests_error_rate_per_endpoint_total{route="GET",server="SERVIDOR-EJEMPLO",app="NombreDeTuServicio",env="development"} 2
```

### Gauge
Este tipo de métrica representa a un valor que puede **incrementar o disminuir** de manera arbitraria. Es muy similiar a *Counter*, con la diferencia de que este valor **sí puede disminuir**. Ejemplos: *Porcentaje de errores por minuto, Núm. de peticiones activas o el uso de memoria de un servicio*.

```
application_httprequests_one_minute_error_percentage_rate{server="SERVIDOR-EJEMPLO",app="NombreDeTuServicio",env="development"} 22.126200053643398
```

### Histograms

Un histograma es una representación que permite realizar la distribución o categorización de un conjunto de datos. Por ejemplo, con ayuda de un histograma podemos saber cuántos valores fueron bajos, cuántos valores fueron medios y cuántos altos.

Histogram se utiliza comunmente para monitorear el tamaño de ciertos eventos. Los casos de uso más comunes es para medir el tiempo de las peticiones o el tamaño de la mismas. Este tipo de métrica toma diferentes observaciones de una métrica en diferentes *momentos u observe* y se colocan en un **Bucket**. Un bucket es una agrupación de información predefinida por nosotros *(para este ejemplo, 10, 20, 50).*

En Prometheus, los buckets se verían de la siguiente manera:
![arquitectura-básica-Prometheus](https://rubenramosdev.github.io/blog/images/post-1/grafica.png#center)

Cada bucket es mayor que el anterior, es decir, sus valores son acumulativos. En este ejemplo, el segundo bucket esta compuesto por la suma de sus valores, más los valores del primero bucket y el tercero, esta compuesto por la suma del segundo y primer bucket.

**_sum** y **_count** por lo general se encuentran presentes en este tipo de métrica. Los 2 campos son valores contadores:
   - _count incrementa 1 con cada observe.
   - _sum se incrementa con el valor de la observación obtenida.

```
application_http_request_duration_seconds_bucket{server="SERVIDOR-EJEMPLO", app="NombreDeTuServicio",env="development", handler="/",le="0.1"} 25688
application_http_request_duration_seconds_bucket{server="SERVIDOR-EJEMPLO", app="NombreDeTuServicio",env="development", handler="/",le="0.2"} 23688
application_http_request_duration_seconds_bucket{server="SERVIDOR-EJEMPLO", app="NombreDeTuServicio",env="development", handler="/",le="0.4"} 28782
application_http_request_duration_seconds_bucket{server="SERVIDOR-EJEMPLO", app="NombreDeTuServicio",env="development", handler="/",le="+Inf"} 29000
application_http_request_duration_seconds_sum{server="SERVIDOR-EJEMPLO", app="NombreDeTuServicio",env="development", handler="/"} 1974.90592036741
application_http_request_duration_seconds_count{server="SERVIDOR-EJEMPLO", app="NombreDeTuServicio",env="development", handler="/"} 29020
```

**le** significa *less than or equal*, es decir, igual o menos. Por lo cuál, esta métrica se puede entender:
-  25688 peticiones tomaron igual o menos de 100 milisegundos (ms).
-  23688 peticiones tomaron igual o menos de 200 milisegundos (ms).
-  28782 peticiones tomaron igual o menos de 400 milisegundos (ms).
- En total han habido 29020 peticiones.
  

### Summary
Esta métrica comparte muchas similitudes con Histogram. Se recomienda su uso en los siguientes escenarios:
- Con los Histograms, los quantiles de una métrica son calculados por Prometheus. Con los Summaries, estos valores son calculados en lado del cliente.
- Si no se tiene una idea clara sobre las definiciones de los buckets o como segmentar un valor, Summary es mejor alternativa, debido a que los histograms necesitan de definiciones de buckets de manera forzosa. 

Recomiendo consultar la [documentación de Prometheus en dónde se aborda el tema a detalle.](https://prometheus.io/docs/practices/histograms/)

## ¿Qué monitorear? ¿Cómo? ¿Cuándo? - The Four Golden Signals
¿Es realmente importante monitorear el nombre de un servidor? O, ¿El nombre de los endpoints? Quizás sí o quizás no, algunas métricas tendrán más sentido para una organización que para otra, lo importante entonces es diseñar un plan de monitoreo en dónde se defina claramente: ¿Qué se va a monitorear y para qué? Es de suma importancia, sobre todo si se estará recopilando información que atañe a diferentes áreas, como por ejemplo, el número de ventas de un producto, el número de fallas de los proveedores externos o el tiempo de respuesta de un determinado endpoint, si bien, estos valores pueden estar relacionados, su información y atención no necesariamente pertenece a la misma área dentro de una organización.

En esta sección me remito de nueva cuenta al artículo Monitoring Distributed Systems [1] en donde se plantean 4 puntos fundamentales que deben ser considerados sí o sí en el monitoreo de nuestros sistemas.

- **Latencia**
  - ¿En cuánto tiempo se están resolviendo las peticiones del cliente? El artículo menciona algo clave, es importante tener cuidado y fijar un sesgo muy claro en este punto, **¡ Los posibles errores que se generen en nuestro sistema no forman parte de la latencia del mismo !** Si nuestro sistema responde en ocasiones con una rapidez de 10ms porque se esta presentando un error, esto no quiere decir que sea rápido 😅. 
- **Tráfico**
  - ¿Cuál es el número de peticiones que esta recibiendo nuestro sistema? ¿Cuál es el promedio por minuto?
- **Errores**
  - ¿Cuántos errores se están generando? ¿Cuál es la tasa de error por minuto?
  
- **Saturación**
  - ¿Qué tanta carga tiene el sistema? ¿Qué tan saturado se encuentra?

Hasta aquí el contenido de esta serie de artículos. En un siguiente artículo se estará cubriendo temas más técnicos de Prometheus como PromQL, el archivo de configuración y Alertmanager, entre otros temas.

## Referencias
> [1] [Monitoring Distributed Systems by Rob Ewaschuk](https://sre.google/sre-book/monitoring-distributed-systems/)   
> [2] [Histograms and summaries](https://prometheus.io/docs/practices/histograms/)   
> [3] [Data model](https://prometheus.io/docs/concepts/data_model/)
