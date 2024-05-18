# Manual para instalar joomla con mysql a partir de una imagen de fedora

## Índice
1. [Introducción](#1-Introducción)
2. [Preparar el Sistema y Descargar la Imagen Base](#2-preparar-el-sistema-y-descargar-la-imagen-base)
3. [Creación del Containerfile y Construcción de Imágenes](#3-creación-del-containerfile-y-construcción-de-imágenes)
4. [Ejecución y Configuración de los Pods](#4-ejecución-y-configuración-de-los-pods)
5[Enlace al vídeo demostrativo](#5-enlace-al-vídeo-demostrativo)

### 1. Introducción

Podman es una herramienta de gestión de contenedores que permite el desarrollo y despliegue de contenedores sin necesidad de un daemon central. Esta práctica final tiene como objetivo profundizar en los conceptos de contenedores mediante la creación, gestión y despliegue de contenedores de aplicaciones usando Podman.

### 2. Preparar el Sistema y Descargar la Imagen Base

#### Instal·lació de Podman
En Fedora, Podman puede ser instalado mediante el siguiente comando:

```bash
   sudo dnf install -y podman
```

#### Verificar la instalación:

```bash
   podman --version
```

#### Descargar la imagen base de Fedora
```bash
   podman pull fedora
```

### 3. Creación del Containerfile y Construcción de Imágenes

Crearemos dos directorios ya que necesitaremos tener dos Containerfile en diferentes ubicaciones.

```bash
   mkdir Joomla
   mkdir Mysql
```

Crearemos el Containerfile para Joomla:
```bash
   nano Joomla/Containerfile
```

Dentro añadiremos el siguiente contenido:

```Dockerfile
FROM fedora:latest

# Instalar Apache, PHP, PHP-FPM y otros paquetes necesarios para Joomla
RUN dnf -y update && \
    dnf -y install httpd php php-mysqlnd php-xml php-json php-gd php-mbstring php-fpm unzip

# Preparar el directorio para el socket de PHP-FPM
RUN mkdir -p /run/php-fpm && \
    chown -R apache:apache /run/php-fpm

# Descargar e instalar Joomla
ADD https://downloads.joomla.org/cms/joomla3/3-9-27/Joomla_3-9-27-Stable-Full_Package.zip /var/www/html/
RUN unzip /var/www/html/Joomla_3-9-27-Stable-Full_Package.zip -d /var/www/html/ && \
    rm /var/www/html/Joomla_3-9-27-Stable-Full_Package.zip

# Copiar archivo de configuración de PHP personalizado
COPY php.ini /etc/php.ini

# Abrir el puerto 80
EXPOSE 80

# Configurar y arrancar Apache y PHP-FPM, asegurando los permisos del socket
CMD ["sh", "-c", "php-fpm && chown apache:apache /run/php-fpm/www.sock && chmod 660 /run/php-fpm/www.sock && /usr/sbin/httpd -D FOREGROUND"]
```

Crearemos el fichero php.ini, es importante crearlo dentro de la carpeta de Joomla:
```bash
nano Joomla/php.ini
```

Y copiamos el siguiente contenido:

```ini
; Maximizar el límite de memoria para scripts PHP
memory_limit = 256M

; Aumentar el tamaño máximo de archivos subidos
upload_max_filesize = 64M

; Aumentar el tamaño máximo de datos POST que PHP aceptará
post_max_size = 64M

; Asegurar que se usen cookies para la sesión en lugar de pasar el ID de la sesión en la URL
session.use_cookies = 1
session.use_only_cookies = 1

; Establecer el manejador de errores para mostrar menos información en producción
display_errors = Off
log_errors = On
error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT

; Establecer zona horaria predeterminada utilizada por todas las funciones de fecha/hora
date.timezone = "Europe/Madrid"

; Aumentar el tiempo máximo de ejecución de scripts PHP
max_execution_time = 300

; Ajustes específicos para mejorar la seguridad
expose_php = Off
```


Haremos lo mismo para el Mysql
```bash
   nano Mysql/Containerfile
```
**Containerfile per MySQL:**
```Dockerfile
# Usar la imagen base de Fedora
FROM fedora:latest

# Instalar MySQL
RUN dnf -y update && \
    dnf -y install mysql-server

# Preparar el directorio de datos
RUN mkdir -p /var/lib/mysql/ && \
    chown mysql:mysql /var/lib/mysql/

# Copiar el script de inicialización
COPY init-mysql.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/init-mysql.sh

# Exponer el puerto 3306
EXPOSE 3306

# Establecer el script de inicialización como comando por defecto
CMD ["init-mysql.sh"]
```

Deberemos crear el script, importante crearlo dentro de la carpeta de Mysql
```bash
   nano Mysql/init-mysql.sh
```

Y copiamos el siguiente contenido
```sh
#!/bin/bash
# Inicializar la base de datos si el directorio está vacío
if [ -z "$(ls -A /var/lib/mysql)" ]; then
    echo "Inicializando la base de datos..."
    mysqld --initialize --user=mysql --datadir=/var/lib/mysql/
    echo "Base de datos inicializada."
fi

# Iniciar el servidor MySQL
echo "Iniciando MySQL..."
exec mysqld --datadir='/var/lib/mysql/' --user=mysql
```

**Construcción de las imágenes:**

Primero construiremos la imagen de Joomla con el nombre myjoomla:
```bash
    podman build -t myjoomla ./Joomla
```
Luego lo mismo con MySQL:
```bash
    podman build -t mymysql ./Mysql
```

### 4. Ejecución y Configuración de los Pods

**Creación del pod y ejecución de los contenedores:**

```bash
    podman pod create --name mypod -p 8080:80 -p 33060:3306
    podman run --pod mypod -it --name myjoomla -d myjoomla
    podman run --pod mypod -it --privileged --name mymysql -d mymysql
```

Primero entraremos al contenedor de MySQL:

```bash
podman exec -it mymysql /bin/bash
```

Una vez dentro, ejecutaremos el siguiente comando para obtener la contraseña temporal que usa MySQL:
```bash
grep "temporary password" /var/log/mysql/mysqld.log 
```

Una vez tengamos esta contraseña, entraremos a MySQL utilizando el siguiente comando:
```bash
mysql -u root -p
```

Dentro de MySQL, ejecutaremos los siguientes comandos:
```bash
ALTER USER 'root'@'localhost' IDENTIFIED BY 'root';
CREATE DATABASE joomla;
FLUSH PRIVILEGES;
```

Después de configurar MySQL, saldremos del contenedor y ejecutaremos el siguiente comando:
```bash
    podman commit mymysql myfinalmysql
```

Ahora entraremos al navegador e ingresaremos a esta URL: http://localhost:8080 para entrar a Joomla, donde debemos completar la siguiente información.
![Joomla config 1](./assets/joomla1.PNG)

En la configuración de la base de datos, ingresaremos los siguientes datos.
![Joomla config 2](./assets/joomla2.PNG)

Podemos observar un error debido a que no existe el archivo solicitado. Ingresaremos al contenedor:
```bash
    podman exec -it myjoomla /bin/bash
```
Ejecutaremos el siguiente comando con el nombre del archivo que ha pedido Joomla:

```bash
    touch /var/www/html/installation/<el nombre de tu fichero>
    chmod 755 /var/www/html/installation/<el nombre de tu fichero>
```

Una vez creado el archivo, volvemos a intentar conectarnos a la base de datos y esta vez obtendremos un resultado satisfactorio. En las siguientes pantallas, podemos dar clic en "Siguiente" sin cambiar nada.

Finalmente, haremos el commit:
```bash
    podman commit myjoomla myfinaljoomla
```

### 5. Enlace al Vídeo Demostrativo

[Inserir enllaç al vídeo]
