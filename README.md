# Contanerización de una aplicación segura en docker

Con esta aplicación de ejemplo se revisará y resaltarán algunos puntos importantes para desplegarla en Docker resaltando la seguridad.

## Funcionamiento de la aplicación

Para saber como desplegar esta aplicación en contenedores se debe conocer su funcionamiento . Consta de dos partes:

- Parte *back* ,formada por un  servicio encargado de  ejecutar una API muy básica definida mediante **node**.

- Parte *front* , que a su vez está formada por dos servicios:
 
   - La parte de entrada a la aplicación, que será un **nginx** para redirigir peticiones y el punto de entrada a la aplicación. 
   - La parte lógica del front. que se comunica con el back  usando **node**.

De modo que desplegaremos dos contenedores contenedores docker (uno para cada parte de la aplicación).

Asociaremos cada servicio definido en cada una de las dos partes con una imagen docker. Por tanto,  debemos crear un fichero Dockerfile por para el back y un fichero Dockerfile *multi-stage*(que contendrá la creación de dos imágenes) para el front.  
Finalmente un fichero *docker compose* para relacionarlos y levantar el escenario.

## Consideraciones de buenas prácticas 

1. ***Elección de imágenes base oficiales***

Para nuestro caso necesitaremos una imagen con node y otra con nginx , de modo que debemos elegirlas del repositorio de imágenes DockerHub teniendo en cuenta que estén verificadas:

![alt text](./images/nginx_oficial.png)

![alt text](./images/node_oficial.png)

2. ***Elección de imágenes base lo más ligeras posible*** 

Para ello , debemos elegir imágenes con el sufijo **slim** , y en la medida de lo posible sistema operativo **alpine**. No solamente es una mejor eficiencia en el uso de recursos destinados al contenedor , sino que además, al reducir el número de paquetes y dependencias , se reducen también el potencial número de vulnerabilidades.

- Para ejecutar la lógica de las aplicaciones solamente es necesario nodejs:

  ![alt text](./images/node_slim.png)

```dockerfile
  FROM node:16.16-slim 
```
- Para la función de proxy necesitamos solamente un nginx con el sistema operativo más ligero:

  ![alt text](./images/nginx_slim_alpine.png)

```dockerfile
  FROM  nginx:alpine3.18-slim
```

3. ***Principio del menor privilegio posible***  

Para evitar potenciales ataques , debemos ejecutar los contenedores sin usuario root(que es el usuario por defecto para los contenedores si no se explicita ninguno).

> Nginx por defecto necesita ejecutarse como root para levantar el proxy salvo que se hagan  ciertas [modificaciones de configuración](https://hub.docker.com/_/nginx).
> Por simplicidad y  siguiendo la [documentación oficial](https://hub.docker.com/r/nginxinc/nginx-unprivileged) , se utilizará una imagen nginx algo más pesada 
> pero con la configuración establecida para poder ejecutar nginx sin privilegios de root.

Por tanto , finalmente usaremos esa imagen para el proxy:

```dockerfile
  FROM  nginxinc/nginx-unprivileged:latest
```
Y para las imágenes node , ejecutaremos el contenedor con un usuario creado mediante la directiva *USER*, otorgando permisos solamente para aquellos directorios donde sea necesario:

```dockerfile
  RUN useradd  usernode && chown -R usernode /app
   ....
  USER usernode
```

4. ***Limitación de los recursos a usar por el contenedor***  

Por defecto , los contenedores que se ejecutan sobre el host no están limitados en cuanto al uso de recursos a utilizar. Se debe evitar esto comportamiento para garantizar la disponibilidad del host donde se ejecutan los contenedores. Para ello, a partir de las versiones 3 de docker compose se debe utilizar la directiva resources dentro de deploy , donde limitamos el uso máximo a utlicar de memoria y de cpu :

```dockerfile
    resources:
        limits:
            cpus: '0.50'
            memory: 512M
        reservations:
            cpus: '0.25'
            memory: 128M  
```
(*) Verificamos con el comando *docker stats*  que efectivamente están limitados los recursos de los contenedores levantados:

|CONTAINER ID |  NAME | CPU %   |  MEM USAGE / LIMIT |  MEM %  |   NET I/O |   BLOCK I/O  | PIDS |
|-------------|---------------|--------|-------------|-----|------------|-----------|--------|
|23d75a5958aa |  docker-nginx-frontend-1 |  0.00% |    9.598MiB / 512MiB |    0.94% | 2.53kB / 0B |  0B / 0B   |  13 |
|ff0b09587fe9 |  docker-nginx-backend-1 |   0.00% |    26.31MiB / 512MiB |  5.14% |    1.32kB / 0B |  0B / 0B   |  7 |


5. ***Apertura de solamante puertos imprescindibles***  

Por seguridad, los puertos de los contenedores están cerrados , salvo que se indique lo contrario al momento de su creación. De modo que solo se deben abrir aquellos puertos que expresamente vayan a ser utilizados. En éste caso solo es necesario habilitar el puerto 8000 para acceder desde el host  al servicio del proxy. Para indicar los puertos a abrir,lo indicamos en el fichero docker compose en la directiva ports  con la siguiente nomenclatura:

> **"NumPortHost:NumPortContainer"**

```dockerfile
    ports:
      - "8000:80"
```

(*) Todos los contenedores definidos en el mismo docker-compose tendrán visibilidad y podrán comunicarse entre ellos. En este caso hemos definido dos subredes: backend y frontend.De no hacerlo, se crearúa una subred automática donde se incluirían los dos contenedores. Sin embargo , es preferible tener el control de las subredes por si fuera necesario  particularizar que  contenedor debe ir a cada subred.

```dockerfile
  networks: 
    frontend:
    backend:
```

6. ***Mejorar la disponibilidad del contenedor***  

En la medida de lo posible , se debe indicar la política a efectuar por docker en caso de fallo del contenedor. En este caso se indica que siempre se vuelvan a levantar en caso de fallo hasta un máximo de tres intentos:

```dockerfile
        restart_policy:
            condition: on-failure
            delay: 5s
            max_attempts: 3
            window: 120s   
```


## Iniciar la aplicación

### Mediante únicamente el fichero  docker compose

Situándonos en el directorio donde se encuentre el *docker-compose.yml* ejecutar lo siguiente:

```dockerfile
   docker-compose up -d --no-build 
```

Donde el valor de cada parámetro:

 *-d* (detached mode) : Ejecutar en modo background. Si se quieren ver los logs del servidor arrancando por consola se debe omitir.

*--no-build* : Evitar la directiva *build* del docker compose ,ya que en este caso nos descargamos las imágenes de docker hub sin necesidad de construirlas.

### Mediante todo el repositorio

Para ello nos debemos descargar el proyecto de git:

Situándonos en la base del proyecto  ejecutar lo siguiente:

```dockerfile
   docker-compose up -d 
```

