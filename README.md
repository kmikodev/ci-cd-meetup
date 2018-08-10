# CI-CD

## Tabla de contenidos 
  - Configuración entorno
  - Arranque de los servicios
  - Configuración
  - Network

## Configuración del entorno
- Instalar docker
    - https://docs.docker.com/install/
  
- Crear el workspace
    - Tomamos de ejemplo el que está en demo:
    ```
    |main
    --------gogs
    |
    --------jenkins
    |       |
    |       --------JENKINS_HOME (dar permisos  sudo chown -R 1000:1000 jenkins/JENKINS_HOME/)    
    --------nexus
    |       |
    |       --------NEXUS_DATA (dar permiso **sudo chown -R 200:200** nexus/NEXUS_DATA/)
    |
    --------nginx
    |       |
    |       --------sites-enable
    |               |
    |               --------gogs.conf
    |               --------jenkins.conf
    |               --------nexus.conf
    |               nginx.conf
    docker-compose.yml
    ```

- Añadir los host a nuestro host local
    - 127.0.0.1 jenkins.ci-cd-meetup.com nexus.ci-cd-meetup.com git.ci-cd-meetup.com 
- Se recomienda instalar la herramienta scope para monitorizar contenedores  https://github.com/weaveworks/scope
## Arranque de los servicios

- Arrancamos los contenedores.
    
    ```
    $   docker-compose up -d
    ```
- Comprobamos el estado de los contenedores con la herramienta que hemos instalado  o con:
    
    ```
    $   docker ps
    ```
Tendríamos que ver los siguientes contenedores

- demo_mysql_1
- demo_gogsl_1
- demo_nexus_1
- demo_gogsData_1
- demo_nginx_1
- demo_jenkins_1


## Configuración

### Nginx

Contenedor que se usará como punto de entrada para exponer los servicios al exterior. Desde aquí podemos exponer tanto por HTTP cómo por HTTPS.

Dispone de los volumenes 
- nginx:/usr/share/nginx/html
- ./nginx/nginx.conf:/etc/nginx/nginx.conf
- ./nginx/sites-enable:/etc/nginx/sites-enable

#### nginx 

Este volumen se usará para alojar las webs estáticas que queramos exponer.

#### ./nginx/nginx.conf

Archivo que usaremos para configurar el servicio que hay dentro del contenedor de nginx

#### ./nginx/sites-enable

Volumen en el que incluiremos los diferentes virtual host 

### Nexus

Nexus será nuestro repositorio de artefactos de todo tipo:

- npm 
- maven
- docker hub (en desarrollo)

Una vez tengamos el servicio arrancado podemos crear los repositorios usando el api de nexus.
  - create-repositories.sh
  - repos.json
  - Ejecución

```bash
#!/bin/bash

jsonFile=$1

printf "Creating Integration API Script from $jsonFile\n\n"
cat $jsonFile

curl -v -u user:pass --header "Content-Type: application/json" 'http://localhost:8081/service/rest/v1/script/' -d @$jsonFile

```

```json
{
  "name": "repositories",
  "type": "groovy",
  "content": "repository.createMavenHosted('maven-library-release'); repository.createMavenHosted('maven-library-snapshot'); repository.createNpmHosted('npm-snapshot'); repository.createNpmHosted('npm-release'); repository.createNpmProxy('npmjs-org','https://registry.npmjs.org'); repository.createNpmGroup('npm-all-pro',['npmjs-org', 'npm-release']); repository.createNpmGroup('npm-all-dev',['npmjs-org','npm-snapshot', 'npm-release'])"
}
```

```bash
$ sh create-repositories.sh repos.json
```

Con el script y el json anterior crearemos repositorios de desarrollo y de producción de npm, y un repositorio proxy que apunta al registry por defecto. Dos grupos uno para desarrollo que contiene el propio de desarrollo (snapshot) y el proxy; otro con el de produccion (release) y el proxy.

#### Habilitar el token para npm.

Nos dirigimos a http://nexus.ci-cd-meetup.com/#admin/security/realms y activamos **npm bearer token realm**


#### ¿Cómo exponer nexus con nginx?

    ```conf
    server {
    listen   80;
    server_name nexus.atresmedia.com;

    # allow large uploads of files
    client_max_body_size 1G;

    # optimize downloading files larger than 1G
    #proxy_max_temp_file_size 2G;

    location / {
        proxy_pass http://10.5.0.4:8081/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    /*
    Con ssl
        location / {
        proxy_pass http://10.5.0.4:8081/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto "https";
        }
    */
    }
    ```

### Gogs

Servicio que vamos a usar para tener un repostorio de fuentes privado. Este repositorio es muy sencillo de configurar y no tiene nada que enviadarle a otras solucciones como pueden ser gitlab.

En la solucción que se está implementando en este caso está compuesto de:

- gogsData: Servicio que contiene los volumenes
- gogs: Es nuestro servicio de git en si
- mysql: Sistema de base de datos que va a acompañar al servicio.

A la hora de instalarlo le indicaremos la ruta y puerto de mysql que en este caso es `10.5.0.8:3306`.

Para configurarlo podemos usar el [cheatset](https://gogs.io/docs/advanced/configuration_cheat_sheet), el que se recomienda para este caso es el siguiente:

```bash
...

[service]
REGISTER_EMAIL_CONFIRM = false
ENABLE_NOTIFY_MAIL     = false
ENABLE_CAPTCHA         = true
REQUIRE_SIGNIN_VIEW    = false
DISABLE_REGISTRATION   = true

...
```

### Jenkins

Jenkins será una de las piezas claves de nuestro sistema de ALM, ya que será el encargado de escuchar los cambios en nuestro repositorio y ejecutar una serie de tareas.

Durante el proceso de instalación nos pedirá un secret-key y procederemos con la instalación.

- Plugins
- Ejemplos de pipeline

Para empezar la instalación tendremos que conectarnos al contenedor por y hacer un cat de **/var/jenkins_home/secrets/initialAdminPassword**

    ```
    $   docker exec -it demo_jenkins_1 bash
    jenkins@d343463cf992:/$ cat /var/jenkins_home/secrets/initialAdminPassword
    386c6a6d10de4699ac1c973ccfaae6a9
    ```

Escoger plugins:
- Junit
- NodeJs

#### Plugins