## Twitter app

## Requerimientos:

Tweets

- Los usuarios deben poder publicar mensajes cortos (tweets) que no excedan un
  límite de caracteres (por ejemplo, 280 caracteres).

Follow:

- Los usuarios deben poder seguir a otros usuarios.

Timeline:

- Deben poder ver una línea de tiempo que muestre los tweets de los usuarios a los
  que siguen.

## Diseño del sistema

El análisis de arquitectura hecho a profundidad se encuentra en el [siguiente](https://docs.google.com/document/d/1tE1kxX1gIPZvUmemteqJIfWkPHaImWT1xwUnUsbLeJc/edit#heading=h.1il4mncsnoch) documento.

Con respecto a la implementación se redujeron casos de uso por quedar fuera de scope. Por lo que se tomaron los casos
base dados en la consigna.

## Explicación del repositorio

El repositorio engloba 3 aplicaciones. Cada aplicación tiene sus propias dependencias.

Todas las aplicaciones se desarrollaron con el lenguaje java y el framework de Spring Boot.

Todas las aplicaciones se desarrollaron utilizando una arquitectura hexagonal para evitar dependencias entre los
módulos, de manera que sea más sencillo mantener y escalar.

Debajo dejo una breve descripción de lo que se hizo en cada aplicación para facilitar una review de lo que se busca
evaluar:

### Users

Servicio encargado de manejar el comportamiento de usuarios y follows:

Casos de uso:

- Persistir follow de usuario.
- Publicar novedad al generarse un nuevo follow por sistema de eventos por colas al persistir.
- Buscar followers de un usuario.

Tecnologías usadas e implemetaciones del repositorio:

- Base de datos H2 (SQL).
- RabbitMQ para publicación de eventos por colas, disparadas al seguir a una persona.
- Implementación de excepciones custom.
- Tests unitarios.
- Tests de integración.

### Tweets:

Servicio encargado de manejar el procesamiento de tweets.

Casos de uso:

- Persistir tweet.
- Generar novedad por sistema de colas a topico de novedades. (el cual escuchará timeline).

Tecnologías usadas e implemetaciones del repositorio:

- Base de dato H2 (SQL) para persistencia
- RabbitMQ para publicación de eventos por colas, disparadas al generar un tweet.

(Aclaración en esta aplicación no se hace validación de usuarios, el tweet que se manda se persiste. Para una
aplicación real y productiva esto debería estar implementado).

### Timeline:

Servicio encargado de procesar el timeline de un usuario.

Casos de uso:

- Consumir novedad de creación de un tweet disparado por el servicio de tweets.
- Persistir la novedad.
- Proveer la información tweets que sigue un usuario.

Tecnologías usadas e implemetaciones del repositorio:

- RabitMq para consumir novedades del topico de novedades de tweets.
- Redis para persistencia.
- Comunicación por http contra Follows para obtener los followers de un usuario.

(Aclaración: se usó redis como persistencia, pero siguiendi el documento de diseño lo ideal es poder mantener una cache
y consultas contra el sistema de tweets para no saturar la cache con usuarios que tienen millones de followers).

### Aclaraciones a nivel general:

- Se utilizó una base de datos H2 en memoria para facilitar el desarrollo y pruebas.

- Muchas de las requests reciben un parametro de **"user_name"** en el header. Esto fue algo que se implementó para el
  ejercicio para no salir del scope, pero la forma de hacerlo sería recibir un token y hacer validaciones contra un
  servicio de
  autenticacion para encontrar los datos de un usuario y validar si las requests y permisios de lo que se quiere hacer
  es valido.

- Se va a visualizar en el código que quedaron variables hardcodeas de puertos, base_urls, contraseñas, etc. Las mismas
  se
  podrian manejar por variables de entorno.

- La aplicación donde se implemetaron tests a detalle fue en la aplicación de users. En el resto no se implementaron por
  un
  tema de tiempos. En los puntos de arriba dejé lo destacable entre ellas para mirar de cada app.

- Como mejora, se va a observar que faltan validaciones en los comportamientos (chequeos de estados, de campos,
  comunicacion entre
  apps), las cuales son importantes para una aplicacion productiva.

- Una mejora para levantar el proyecto es dockerizar las 3 aplicaciones con sus dependencias.

- Otra mejora es usar liberías de linters y jenkins para correr los mismos en CI.

- Hay casos de uso border que se indicaron en el documento que no estan implementados en el código.

## Endpoints:

### Seguir a alguien:

curl --location --request PUT 'localhost:8080/follows/follow' \
--header 'user_name: charly' \
--header 'Content-Type: application/json' \
--data '{
    "followee": "mariano"
}'

### Ver los usuarios seguidos de un usuario:

curl --location --request GET 'localhost:8080/follows/followees?offset=0' \
--header 'user_name: charly' \
--header 'Content-Type: application/json' \
--data '{
    "followee": "mariano"
}'

### Ver los followers de un usuario:

curl --location --request GET 'localhost:8080/follows/followers/mariano?offset=0' \
--header 'user_name: charly' \
--header 'Content-Type: application/json' \
--data '{
    "followee": "mariano"
}'

### Publicar un tweet

curl --location 'localhost:8091/tweet' \
--header 'user_name: mariano' \
--header 'Content-Type: application/json' \
--data '{
    "content": "hola!"
}'


### Ver el timeline de un usuario

curl --location 'localhost:8090/timeline?fromIndex=0' \
--header 'user_name: charly' \
--data ''


### Como levantar el proyecto

(Aclaración, Idealmente las 3 apps deberian levantarse con unico comando de docker-compose).

1- Tener instalado docker.
2- Bajar imagen de rabbitmq y redis. (latests)
3- Configurar rabbit mq agregando los topicos de tweets-events-queue y follows-events-queue:
  - desde http://localhost:15672. El login es username: guest, password: guest.
4- Aregar los exchange de tweets-events-exchange y follows-events-exchange, y agregar los bindings a sus topicos. (Esto tambien se podria sumar en una property):

Agregar topicos y exchanges:

-  http://localhost:15672/queues -> en la sección de add new agregar el nombre **follow-events-queue** y hacer click en add queue.
-  http://localhost:15672/queues -> en la sección de add new agregar el nombre **tweet-events-queue** y hacer click en add queue.
-  http://localhost:15672/exchange -> en la sección de add new agregar el nombre **follow-events-exchange** y hacer click en add exchange.
-  http://localhost:15672/exchange -> en la sección de add new agregar el nombre **tweet-events-exchange** y hacer click en add exchange.

Agregar bindings:

desde http://localhost:15672/exchange en la tabla hacer click en el nombre de los exchange creados. Esto llevara a una vista para agregar los bindings (conexion topicos con los exchange).

follow-events-exchange:

- Add binding from this exchange -> To queue -> agregar el topico de follow-events-queue, como routing key dejar follow-events-routing-key

tweet-events-exchange:

- Add binding from this exchange -> To queue -> agregar el topico de tweet-events-queue, como routing key dejar tweet-events-routing-key

5- Desde la app user correr el comando docker-compose up. (con esto se levantará rabbitmq).
6- Levantar aplicacion de users.
7- Levantar aplicación de tweets.
8- Correr comando docker-compose up en aplicación de timeline.
9- Levantar aplicación de timeline.
