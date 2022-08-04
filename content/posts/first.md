---
title: "Un primer vistazo a Prometheus."
url: "/posts/promethus-basics-1"
summary: "Introducción, conceptos clave y explicación del funcionamiento de esta herramienta para el monitoreo de servicios."
tags: ["métricas", "prometheus", "microservicios", "monitoreo"]
cover:
    image: images/post-1/cover.jpg
    alt: ''
    caption: ''
---

# Prometheus, ¿Qué es?
Imaginameos que contamos con 200 microservicios desplegados en nuestro cluster, cada microservicios genera cierta cantidad de tráfico, algunos pueden ser más críticos para nuestra operativa que otros y cada uno genera información que nos puede ser relevante.

Ahora bien, ¿Qué podemos hacer para conocer su estado? ¿Conocer  ¿Cómo conocerel número de peticiones que reciben? O, ¿el porcentaje de error de cada servicio? Si no estamos familizarizados con el tema de monitoreo, quizás la primer idea que venga a nuestra mente es tratar de obtener esta información a través de los logs de la aplicación. No obstante, existe una mejor alternativa, existe Prometheus.

Prometheus es un **sistema de monitoreo de código abierto** cuyo principal función se centra en  **recompilar, almacenar y exponer** la información de un conjunto de servicios. 

Si bien, en esta serie de artículos nos vamos a centrar en el monitoreo de microservicios, Prometheus puede monitorear **cualquier servicio capaz de exponer sus métricas**, siempre y cuando, esta información tenga el **formato específico** requerido por Prometheus. Por ejemplo, se pueden monitorear servidores Linux, Windows, NGINX, Kafka o bases de datos como MySQL, SQL Server, etc...

Este es un ejemplo del tipo de resultados que podemos obtener con un sistema de monitoreo conformado con Promethus y Grafana
![ejemplo-métricas](https://miro.medium.com/max/1312/1*dzODOIDqXTsWauf5eM8Trw.gif)

## ¿Cómo funciona?, Entendiendo su arquitectura
La siguiente imagen muestra la arquitectura de Promethus de forma muy general, no obstante, nos ayuda a comprender como funciona internamente. 

Como mencionado anteriormente, Promethus nos ayuda a **recompilar, almacenar y exponer** información. Para entender cómo es el flujo de trabajo, veamos la siguiente imagen:
![arquitectura-básica-promethus](/images/post-1/promethus-architecture.png)

1. **Retrival**: Es un motor de *scrapping* encargado de **recolectar** la información de los diferentes servicios *(conocidos como Jobs)* que nos interesa monitorear. En un intervalo de tiempo definido por nosotros por ejemplo, cada 20 segundos, 2 minutos. o 5 min. el scrapper se encargará de consultar los servicios registrados con el objetivo de recolectar su información.   
Los intervalos de tiempo pueden ser generales para todos los *Jobs* o cada *Job* puede tener un tiempo particular.
1. **Time series database o TSDB**: Una vez que la información ha sido recolectada, Promethus **almacena** la misma en una base de datos de tipo **Time series** *(se explica más adelante)*.   
De forma nativa, Promethus ya cuenta con la integración de una base de datos para almacenar la información de forma local, en disco. No obstante, se pueden integrar alternativas para almacenar la información de forma remota.
1. **HTTP Server**: Promethus cuenta con un servidor HTTP integrado. Este servidor **expone** la información previamente recolectada y almacenada por medio de una ruta http, */metrics*. Con la información publicada de esta forma, se pueden utilizar múltiples aplicaciones especializadas para la creación de gráficos, como puede ser Grafana para mostrar la información de una manera más entendible para los seres humanos.  

## Métricas, ¿Qué son?
En el sitio oficial de Prometheus, encontramos la siguiente definición.
> In layperson terms, metrics are numeric measurements. Time series means that changes are recorded over time. 

Sin más, **una métrica es un medición numérica.**

No obstante la definición va más allá de eso y nos comienza a introducir un concepto clave de Promethus y es **Time series**, en español **Serie Temporal**. De manera general, una serie temporal es una sucesión de datos medidos en determinados momentos y ordenados cronológicamente, con los mismos valores y los mismos identificadores *(Tags)*.

Si unimos estas dos definiciones, podemos entender el funcionamiento del almacenamiento de Promethus, quién fundamentalmente recolecta, ordena y almacena la información de manera cronológica, obteniendo un conjunto de datos que están cambiado con el tiempo.

Por ejemplo, la siguiente imagen es una animación del tipo de gráfica que se puede lograr con esta información:
![gráfica-con-datos-series-temporal](https://media.giphy.com/media/rM0wxzvwsv5g4/giphy.gif)

Gracias a esta forma de organizar la información, tenemos posibilidad de ver las tendencias o comportamiento de ciertos valores, por ejemplo, el porcentaje de ventas en cierto período de tiempo o el nivel de carga de nuestros servicios en diferentes ventanas de horarios, etc... 

### Formato de las métricas, ¿Cómo esta compuesta una métrica?

Ahora que ya tenemos un contexo más amplio acerca de Prometheus, es momento de entrar en detalles más técnicos. Como mencionado anteriormente, Promethus puede recolectar la información de múltiples servicios, poco importa si es un microservicio, un servidor o una BD, la única condición es que esta información cuente con un formato específico para que pueda ser interpretada, recolectada y almacenada eventualmente.

En la documentción oficial de Promethus, nos encontramos el siguiente ejemplo,

```
<nombre de la métrica>{<nombre etiqueta>=<valor de la etiqueta>, ...}
```

Y así, se ve un ejemplo real de métrica:
```
application_httprequests_active{server="SERVIDOR-EJEMPLO"} 1
```
El anterior ejemplo se trata de una métrica cuyo objetivo es dar a conocer el número de peticiones que se encuentran activas en el servidor. 

De manera desglosada, podemos analizar cómo se compone una métrica:

- **application_httprequests_active**: Nombre de la métrica.
- **server=SERVIDOR-EJEMPLO**: Este valor es conocido como una **etiqueta o label**. Una etiqueta es información extra para precisar o identificar de mejor manera el origen de la información, adelante veremos más ejemplos.  
- **1** : Valor de la métrica.

En conclusión, **en el servidor de nombre "SERVIDOR-EJEMPLO" se encuentra 1 petición activa en el momento en que fue consultado el valor.**

## Tipos de métricas
Prometheus implementa 4 tipos de métricas diferentes para almacenar su información.

### Counter
Counter representa a el tipo de medición cuyo valor es solamente acumulativo, es decir, **incrementa y no disminuye**. Ejemplos: *Número de peticiones a un servicio, núm. de errores o las ventas de algún producto.*
```
application_httprequests_error_rate_per_endpoint_total{route="GET",server="SERVIDOR-EJEMPLO",app="NombreDeTuServicio",env="development"} 2
```

### Gauge
Este tipo de métrica representa a un valor que puede **incrementar o disminuir** de manera arbitraria. Es muy similiar a *Counter*, con la diferencia de que este valor sí puede disminuir. Ejemplos: *Porcentaje de errores por minuto, Núm. de peticiones activas o el uso de memoria de un servicio*.

```
application_httprequests_one_minute_error_percentage_rate{server="SERVIDOR-EJEMPLO",app="NombreDeTuServicio",env="development"} 22.126200053643398
```

### Histogram
Propongo el siguiente ejemplo para entender este tipo de métrica de la mejor manera. Imaginemos que necesitamos monitorear el porcentaje de errores de nuestra aplicación pero en diferentes momentos, es decir, queremos saber cuál es el porcentaje de error en 1 min, 5 min, 30 min y 60 min. 

¿Cómo hacerlo? Con lo revisado hasta el momento, podríamos crear 4 métricas indivudales de tipo *Gauge* para monitorear estos valores. El problema de este aproximación es que duplicaríamos 4 métricas cuya única diferencia es el momento en cuál fueron registradas.

Bueno, *Histogram* es la respuesta correcta a nuestro planteamiento. Este tipo de métrica toma diferentes observaciones de una métrica en diferentes *momentos*. Promethus se encarga de recolectar esta información en diferentes **Buckets**, que son agrupraciones de información predefinidas por nostros, de esta manera evitamos duplicar una métrica N veces.

```

```

### Summary
Esta métrica es muy similar a Histogram. Solo que, además de recolectar valores y establecerlos en Buckets *(agrupraciones de información predefinidas por nostros)*, también genera la suma de los valores registrados y el número de registros que se han realizado de esta métrica.
```
application_httprequests_transactions_sum{server="SERVIDOR-EJEMPLO",app="NombreDeTuServicio",env="development"} 1.8582496999999998
application_httprequests_transactions_count{server="SERVIDOR-EJEMPLO",app="NombreDeTuServicio",env="development"} 36
application_httprequests_transactions{server="SERVIDOR-EJEMPLO",app="NombreDeTuServicio",env="development",quantile="0.5"} 0.0005997
application_httprequests_transactions{server="SERVIDOR-EJEMPLO",app="NombreDeTuServicio",env="development",quantile="0.75"} 0.0073135
application_httprequests_transactions{server="SERVIDOR-EJEMPLO",app="NombreDeTuServicio",env="development",quantile="0.95"} 0.2358127
application_httprequests_transactions{server="SERVIDOR-EJEMPLO",app="NombreDeTuServicio",env="development",quantile="0.99"} 0.4829384
```
## ¿Qué minitoriear? ¿Cómo? ¿Cuándo?. Reglas para un monitoreo exitoso sin morir en el intento
Whitebox monitoring
Tags
Lenguaje de Prometheus
Exporter
Tipos de exportación