# Contanerización de una aplicación segura en docker

Con esta aplicación de ejemplo se revisará y resaltarán algunos puntos importantes para desplegarla en Docker resaltando la seguridad.

## Funcionamiento de la aplicación

Para saber como desplegar esta aplicación en contenedores se debe conocer su funcionamiento . Consta de dos partes:

- Parte *back* ,formada por un  servicio encargado de  ejecutar una API muy básica definida mediante **node**.

- Parte *front* , que a su vez está formada por dos servicios:
 
   - La parte de entrada a la aplicación, que será un **nginx** para redirigir peticiones y el punto de entrada a la aplicación. 
   - La parte lógica del front. que se comunica con el back  usando **node**.

De modo que desplegaremos dos contenedores contenedores docker (uno para cada parte de la aplicación).

Asociaremos cada servicio definido en cada una de las dos partes con una imagen docker. Por tanto,  debemos crear un fichero Dockerfile  *multi-stage* para el back y un fichero Dockerfile *multi-stage* para el front. 


#### Dockerfile backend

```dockerfile
FROM node@sha256:e9dbce470d22c34e5cc91e305f5ad3fd14b3f02e36fd8a7746e3e5a9e4de4655 AS build-env

WORKDIR /app
RUN groupadd -g 1002  nodegroup && useradd -u 1002 -g 1002 usernode && chown -R usernode /app && chmod 550 /app

COPY ./package.json /app/package.json
COPY ./package-lock.json /app/package-lock.json

RUN npm ci
RUN chmod 770 /app/node_modules
USER usernode
COPY . .

CMD ["node", "server.js"]

# Image node distroless(neither user privilegies nor shell )
FROM gcr.io/distroless/nodejs@sha256:0f7fbe2f3853fd719204ff417dda421eea2f8db8e17875820fddac5d3a8f572c
COPY --from=build-env --chown=1002:1002 /app /app
COPY --from=build-env /etc/passwd /etc/passwd
COPY --from=build-env /etc/group /etc/group

USER 1002

WORKDIR /app
CMD [ "server.js"]
```
#### Dockerfile frontend

```dockerfile
#Image node with yarn for building the front
FROM node@sha256:e9dbce470d22c34e5cc91e305f5ad3fd14b3f02e36fd8a7746e3e5a9e4de4655 AS build
ARG REACT_APP_SERVICES_HOST=/services/m

WORKDIR /app

COPY ./package.json /app/package.json
COPY ./package-lock.json /app/package-lock.json
RUN yarn install

COPY . .
RUN yarn build

#Image debian for creating the nginx configuration with no root
FROM debian@sha256:2c22645bfe97aa1ed1c930adf5970fee3454f9a42a19214051ec677cba805712 as builder

ENV USER=nonroot
ENV UID=10001 

RUN adduser \    
    --disabled-password \    
    --gecos "" \    
    --home "/nonexistent" \    
    --shell "/sbin/nologin" \    
    --no-create-home \    
    --uid "${UID}" \    
    "${USER}"

RUN apt update && apt -y install wget gnupg binutils && \
    wget "https://nginx.org/keys/nginx_signing.key" && \
    apt-key add nginx_signing.key && \
    echo "deb https://nginx.org/packages/mainline/debian/ bullseye nginx" >/etc/apt/sources.list.d/nginx.list && \
    echo "deb-src https://nginx.org/packages/mainline/debian/ bullseye nginx" >>/etc/apt/sources.list.d/nginx.list && \
    apt update && \
    apt install -y nginx && \
    apt install --download-only --reinstall nginx lsb-base  libgcc-s1 libc6 libcrypt1 libpcre2-8-0 libssl1.1 zlib1g && \
    apt install --download-only --reinstall libidn2-0 libnss-nis libnss-nisplus  debconf gcc-10-base && \
    for f in /var/cache/apt/archives/*.deb; do dpkg-deb -xv $f /packages; done && \
    rm -rf /packages/usr/share/bash-completion  \
        /packages/usr/share/debconf \
        /packages/usr/share/doc \
        /packages/usr/share/lintian	\
        /packages/usr/share/locale	\
        /packages/usr/share/man \
        /packages/usr/share/perl5  \
        /packages/usr/share/pixmaps \
        /packages/usr/bin \
        /packages/usr/bin/dpkg* \
        /packages/usr/bin/nginx-debug && \
        mkdir -p /packages/var/run

RUN chown -R nonroot:nonroot /var/cache /var/run

RUN touch /var/run/nginx.pid && \
    chown -R nonroot:nonroot /var/run/nginx.pid

# Empty image with the result of the nginx and node images and entrypoint of the app

FROM scratch
COPY --from=builder /packages /
COPY --from=builder /etc/passwd /etc/passwd
COPY --from=builder /etc/group /etc/group
COPY --from=builder /var/cache /var/cache
COPY --from=builder /var/run /var/run

COPY ./nginx/nginx.conf /etc/nginx/nginx.conf
COPY ./nginx/default.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/build /usr/share/nginx/html

RUN --mount=type=bind,from=builder,target=/mount ["/mount/bin/ln", "-sf", "/dev/stdout", "/var/log/nginx/access.log"]
RUN --mount=type=bind,from=builder,target=/mount ["/mount/bin/ln", "-sf", "/dev/stderr", "/var/log/nginx/error.log"]

EXPOSE 8000

USER nonroot

CMD ["/usr/sbin/nginx", "-g", "daemon off;"]
```

Finalmente un fichero *docker compose* para relacionarlos y levantar el escenario.

#### Docker compose 

```dockerfile
version: '3.8'

services:
  frontend:
    image: jascarr/exercise-nginx-frontend:v1 
    build: 
      context: ./frontend
      args:
        - REACT_APP_SERVICES_HOST=/services/m
    deploy:
        restart_policy:
            condition: on-failure
            delay: 5s
            max_attempts: 3
            window: 120s
        resources:
            limits:
              cpus: '0.50'
              memory: 512M
            reservations:
              cpus: '0.25'
              memory: 128M        
    ports:
      - "8000:80"
    networks: 
      - frontend
      - backend
  
  backend:
    image: jascarr/exercise-node-backend:v1 
    build:
      context: ./backend
    deploy:
        restart_policy:
            condition: on-failure
            delay: 5s
            max_attempts: 3
            window: 120s    
        resources:
            limits:
              cpus: '0.50'
              memory: 512M
            reservations:
              cpus: '0.25'
              memory: 128M      
    networks: 
      - backend

networks: 
  frontend:
  backend:
```

## Consideraciones de buenas prácticas 

1. ***Elección de imágenes base oficiales***

Para nuestro caso necesitaremos una imagen con node y otra con nginx , de modo que debemos elegirlas del repositorio de imágenes DockerHub teniendo en cuenta que estén verificadas:

![alt text](./images/nginx_oficial.png)

![alt text](./images/node_oficial.png)

> (*) Adicionalmente , se debe configurar Docker para que solamente interactúe con imágenes 
> verificadas . Para ello se debe cambiar el valor de la variable  **DOCKER_CONTENT_TRUST** a 1.
> Windows(Powershell): **$env:DOCKER_CONTENT_TRUST=1**
> Linux: **export DOCKER_CONTENT_TRUST=1**

2. ***Elección de imágenes base lo más ligeras posible*** 



  De esta forma, no solamente es una mejor eficiencia en el uso de recursos destinados al contenedor , sino que además, al reducir el número de paquetes y dependencias , se reducen también el potencial número de vulnerabilidades.


  El funcionamiento de docker para definir varias imágenes en un mismo Dockerfile(llamado así multi stage), funciona de tal manera que solamente permanece en la imagen final la última imagen definida en Dockerfile , creándose las anteriores de forma temporal , permitiendo usarlas por ejemplo para crear un artefacto resultado de una compilación, sin necesidad de almacenar todo ese software adicional que no será necesario en adelante, reduciendo así la superficie de ataque. 

  De manera ordenada de más a menos prioritaria, dependiendo de la naturaleza de nuestra aplicación, debemos crear las imágenes en el Dockerfile en el siguiente orden:

    1. ***Imágenes vacías(scratch)***

    Son imágenes sin ningún tipo de software adicional. Por ejemplo en el caso de nuestro Dockerfile *frontend*, la imagen final será muy poco pesada ya que la tercera y última imagen está formada por una imagen vacía:

    ```dockerfile
      FROM scratch
    ```
    Donde posteriormente copiamos todo lo necesario generado en las dos imágenes anteriores.

    2.  ***Imágenes distroless***

    Son aquellas imágenes que serán usadas en entornos de producción, especialmente en pods en kubernetes, ya que no contienen muchos de los paquetes que se encuentran incluso en los sistemas operativos más ligeros, como /bin/sh o /bin/bash. Esto se traduce en que no contienen funcionalidades como la execución de una shell. Además tampoco cuentan para ejecutarse por defecto con un usuario con privilegios de administrador. Esto las hace muy eficientes y seguras.
    Por ejemplo para la imagen final en el backend , se utiliza una imagen de esta clase:

    ```dockerfile
      FROM gcr.io/distroless/nodejs
    ```

    3.  ***Imágenes alpine-slim***

    En aquellos entornos de pruebas donde debamos interactuar dentro del contenedor o porque nuespara la lógica de nuestra aplicación no haya disponible una imagen de los tipos anteriores, debemos elegir imágenes con el sufijo **slim** , y en la medida de lo posible sistema operativo **alpine**. 

    Posteriormente , para entornos productivos podríamos eliminar las funciones como /bin/bash o ejecutar las imágenes con un usuario y grupo creado otrogando los permisos imprescindibles.
    


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



> En linux  , los permisos de los directorios y ficheros se gestionan de más a menos prioritarios de  
> la siguiente manera:
>> - propietario del fichero o directorio
>> - grupo al que pertenece el propietario del fichero o directorio
>> - resto de usuarios
>
> Además a cada usuario y al grupo al que pertenece se le asocia un identificador:
>> - uid: identificador numérico de usuario. Por convención comienzan a partir de 1000.
>> - gid: identificador numérico de grupo.Por convención comienzan a partir de  1000.
>
> La información de los usuarios y sus grupos se encuentra alojada en los ficheros */etc/passwd* y 
> *etc/group*. Por tanto para mantener los permisos de una imagen a otra debemos copiar estos   
> ficheros como hacemos en el dockerfile frontend , durante la creación de la última imagen:
>
> ```dockerfile
>  COPY --from=builder /etc/passwd /etc/passwd
>  COPY --from=builder /etc/group /etc/group
> ```

Teniendo esto en cuenta, para mantener el control de los mínimos permisos que necesitan los usuarios, se debe crear un usuario y un grupo  para ese usuario al que le debemos asignar el uid y el gid respectivamente. Además solamente debe tener permisos para las carpetas y ficheros imprescindibles, tal  y como se realiza en la primera imagen node definida en el Dockerfile backend:

```dockerfile
RUN groupadd -g 1002  nodegroup && useradd -u 1002 -g 1002 usernode \ 
&& chown -R usernode /app && chmod 550 /app && chmod 770 /app/node_modules
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

En la medida de lo posible , se debe indicar la política a efectuar por docker en caso de fallo del contenedor. En este caso se indica que siempre se vuelvan a levantar en caso de fallo hasta un máximo de tres intentos en el docker compose:

```dockerfile
        restart_policy:
            condition: on-failure
            delay: 5s
            max_attempts: 3
            window: 120s   
```
Además en el Dockerfile se puede añadir la directiva  [*HEALTHCHECK*](https://docs.docker.com/engine/reference/builder/#healthcheck), para monitorizar el estado del contenedor y ejecutar peticiones al endpoint del contenedor para asegurarnos que funciona correctamente:

```dockerfile
    HEALTHCHECK --interval=5m --timeout=3s \
    CMD curl -f http://localhost || exit 1 
```

O bien añadirlo al docker compose:

```dockerfile
    healthcheck:
      test: curl --fail http://localhost || exit 1
      interval: 5s
      timeout: 3s
      retries: 1
      start_period: 2s  
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

