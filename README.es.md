<h1>
  <p align="center" width="100%">
    <img width="44%" src="../.recursos/img/logos/nextcloud.png">
    </br>
    Nextcloud
  </p> 
</h1>

<h2> 
  <p align="center" width="100%">
    Un nube de almacenamiento personal combinada con una suite de ofimática colaborativa.
  </p>
</h2>

<h3> 
  <p align="left" width="100%">
    Basado en la suite de <a href="https://nextcloud.com">Nextcloud</a> y contenerizado por <a href="https://linuxserver.io">Linuxserver.io</a>
</h3>

[![Static Badge](https://img.shields.io/badge/lang-%F0%9F%87%AC%F0%9F%87%A7_en-blue?style=plastic)](README.md)

<h3>
  Contenido:
</h3>

- [Estructura](#estructura)
- [Descripción](#descripción)
  - [*Otras observaciones*](#otras-observaciones)
  - [*Variables de entorno*](#variables-de-entorno)
  - [*Antes de empezar*](#antes-de-empezar)
- [Arranque del contenedor y configuración inicial](#arranque-del-contenedor-y-configuración-inicial)

## Estructura

    nextcloud/
      ├─ docker-compose.yml               → archivo docker
      ├─ .env                             → variables de entorno
      ├─ data/                            → carpeta de datos de Nextcloud
      ├─ mariadb/                         → carpeta de datos de MariaDB (base de datos)
      ├─ config/
      │    └─ log/
      │         └─ nginx/                 → logs de nginx (ver más adelante)
      └─ redis/
           └─ sysctl.conf                 → configuración de redis (ver más adelante)

## Descripción

Los archivos `docker-compose.yml` y `.env` como siempre no necesitan presentación, son los archivos que contienen todas las instrucciones y variables para crear el contenedor de Nextcloud.

La carpeta `data/` se creará junto al primer arranque del contenedor y mantendrá los datos de la aplicación una vez en marcha. No necesitamos crearla manualmente. Entre todos los datos se generará un archivo `nextcloud.log` que luego podremos pasar a `crowdsec` (ver en el contenedor dedicado).

La carpeta `mariadb/` al igual que la anterior se creará por si sola durante el primer arranque. Mantendrá la base de datos relacional en la que se apoya esta implementación de Nextcloud. Existen otras bases de datos que podrían usarse, pero esta es la más común para un uso no intensivo.

La carpeta `config/` es otra carpeta en la que no tenemos que hacer nada. El único motivo por el que se mapea a una carpeta del servidor es para tener acceso a la carpeta de logs de Nginx, y así estarán disponibles para ser leídos por `crowdsec` (ver en la [sección dedicada](../crowdsec/)).

Por último, la carpeta `redis/` contiene el archivo de configuración `sysctl.conf` cuyo único propósito es añadir una línea de código para evitar que en un supuesto escenario de baja memoria se produzca un fallo en el volcado de los datos de la misma. Al igual que el resto de carpetas de éste servicio su inclusión es opcional.

En resumen:

  * **Toda la estructura de carpetas es completamente opcional**. Podemos poner Nextcloud en servicio sin necesidad de crear ningún punto de montaje al sistema anfitrión.
  
  * Sólo se han añadido dichos puntos de montaje para tener un acceso más fácil a los registros y a ciertos archivos de configuración puntuales. La mayoría de casos no necesitan de ello.

### *Otras observaciones*

Dentro de `docker-compose.yml` hay que hacer un par de aclaraciones:

  * En la sección `environment:` tenemos que ajustar manualmente `TRUSTED_PROXIES` con la dirección de la red `proxy`. La red fue creada durante la configuración de [Traefik](../traefik/README.md). Podemos ver el dato ejecutando lo siguiente:

```bash
docker network inspect proxy | grep "Subnet"
```

  * También en el contenedor Traefik hay definido un [middleware](../traefik/rules/middlewares.yml) para las cabeceras de Nextcloud. Puedes consultar allí la configuración, o ajustarla a tus necesidades.

  * En la sección `volumes:` podemos añadir todas las carpetas que queramos como por ejemplo un almacenamiento externo. Generalmente se mapean bajo las carpetas `/srv` o `/mnt` dentro del contenedor. **Para poder hacer uso de ellas tendremos que habilitar el plugin "Almacenamiento externo" dentro de Nextcloud.**

Parte del supuesto de que va a ser accesible desde fuera de nuestra red local. Por lo tanto hay que generar el correspondiente CNAME en nuestro proovedor DNS.

Se apoya en **MariaDB** para la base de datos. Aunque se puede configurar con SQLite, a la larga tener una base de datos dedicada da mejor resultado.

Se apoya en **Redis** para mantener la caché en memoria. Lo recomiendo incluso para entornos de un solo usuario porque la mejora de rendimiento es palpable.

### *Variables de entorno*

* `PUID` y `PGID` son los identificadores de usuario y grupo en formato numérico (ejecutar `id` para conocerlos)
* `TZ` es la zona horaria en formato `Continente/Ciudad`. [Listado de zonas](https://www.joda.org/joda-time/timezones.html)
* `DOCKERDIR` es el directorio que contiene todos los servicios de Docker.
* `DOMAINNAME` es el nombre de nuestro dominio.
* `DB_PASSWORD` y `DB_ROOT_PASSWORD` son las contraseñas necesarias para MariaDB. Elegir dos buenas contraseñas es importante.

### *Antes de empezar*

* Como se ha explicado anteriormente, este servicio es dependiente de la red `proxy` para acceder desde el exterior. Ajustar `TRUSTED_PROXIES` en `docker-compose.yml`.

* Crear un registro CNAME en nuestro proovedor DNS. En `docker-compose.yml` aparece como `nextcloud.$DOMAINNAME`, aunque lo podemos llamar de la forma que prefiramos.

* Crear la carpeta `redis/` y el archivo `sysctl.conf` con su contenido.

## Arranque del contenedor y configuración inicial

```bash
docker compose up -d       → arrancamos Nextcloud en segundo plano

docker logs nextcloud -f   → examinamos los registros para ver si hay algún problema (CTRL+c para salir)
```
</br>

En un navegador vamos a la dirección que hayamos configurado (`https://nextcloud.ejemplo.com`)

Se mostrará el asistente de configuración inicial. Para ello:

  1. Introducir el nombre del usuario que será administrador.
  2. Introducir la contraseña.
  3. (Opcional) Instalar las aplicaciones recomendadas ahora.
  4. Hacer clik en Almacenamiento y base de datos.

</br>
  <p align="center" width="100%">
    <img width="33%" src="../.recursos/img/nextcloud/nextcloud_wizard_01.png">
  </p>
</br></br>

  5. Seleccionar la base de datos que queramos utilizar. En este caso, **MariaDB**
  6. Usuario: **`nextcloud`**
  7. Contraseña: **`DB_PASSWORD`** (configurada en `.env`)
  8. Nombre de la base de datos: **`nextcloud`**
  9. Host: **`nextcloud-db:3306`**

  <p align="center" width="100%">
    <img width="33%" src="../.recursos/img/nextcloud/nextcloud_wizard_02.png">
  </p>
</br></br>

Accedemos con las credenciales:

  <p align="center" width="100%">
    <img width="33%" src="../.recursos/img/nextcloud/nextcloud_login.png">
  </p>

<h3>
¡Listo! Ya tenemos una nube personal con acceso a docenas de aplicaciones.
</h3>