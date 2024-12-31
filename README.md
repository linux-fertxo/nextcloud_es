<h1>
  <p align="center" width="100%">
	<img width="44%" src="../.recursos/img/logos/nextcloud.png">
	</br>
	  Nextcloud
  </p> 
</h1>

<h2> 
  <p align="center" width="100%">
  	A self-hosted cloud storage combined with a collaborative office suite.
  </p>
</h2>

[![Static Badge](https://img.shields.io/badge/lang-%F0%9F%87%AA%F0%9F%87%B8_es-blue?style=plastic)](README.es.md)

<h3> 
  <p align="left" width="100%">
	  Based on the <a href="https://nextcloud.com">Nextcloud</a> suite and containerized by <a href="https://linuxserver.io">Linuxserver.io</a>
  </p>
</h3>

<h3>
  Content:
</h3>

- [Structure](#structure)
- [Description](#description)
  - [*Other remarks*](#other-remarks)
  - [*Environment Variables*](#environment-variables)
  - [*Before starting*](#before-starting)
- [Starting the container and initial configuration](#starting-the-container-and-initial-configuration)

## Structure

    nextcloud/
      ├─ docker-compose.yml               → dockerfile
      ├─ .env                             → environment variables
      ├─ data/                            → Nextcloud data folder
      ├─ mariadb/                         → MariaDB data folder (database)
      ├─ config/
      │    └─ log/
      │          └─ nginx/                → nginx logs (see below)
      └─ redis/
           └─ sysctl.conf                 → redis configuration (see below)

## Description

The `docker-compose.yml` and `.env` files, as always, need no introduction. These are the files that contain all the instructions and variables to create the Nextcloud container.

The `data/` folder will be created in the first run and will hold the application data once it's running. We don't need to create it manually. Among all the data, a `nextcloud.log` file will be generated which we will then pass to `crowdsec` (see in the [dedicated section](../crowdsec/)).

The `mariadb/` folder, like the previous one, will be generated during the first run. It will hold the relational database on which this Nextcloud implementation relies. There are other databases that could be used, but this is the most common one for a non-intensive use (SQLite falls short and PostgreSQL is too much).

The `config/` folder is another folder where we don't have to do anything. The only reason it's mapped here to a physical folder on the server is for having access to the Nginx logs, so they will be available for `crowdsec` (see in the [dedicated section](../crowdsec/)).

Finally, the `redis/` folder contains the `sysctl.conf` configuration file, whose sole purpose is to add a line of code to prevent in a supposed low memory scenario from causing a failure in the dump of the data. As with the rest of the folders of this service, its inclusion is optional.

>**In brief:**
>* **The entire folder structure is completely optional**. We can run Nextcloud without creating any mount points on the host system.
>* These mount points have only been added for easier access to logs and certain configuration files. But **most cases won't require it.**

### *Other remarks*

Within `docker-compose.yml` a few clarifications:

* In `environment:` section you have to manually set `TRUSTED_PROXIES` to the `proxy` network address. The network was created during the configuration of [Traefik](../traefik/README.md). You can find out by running the following command:

```bash
docker network inspect proxy | grep "Subnet"
```
>

* Also in the Traefik container there is a [middleware](../traefik/rules/middlewares.yml) defined for the Nextcloud headers. You can check the configuration there and adjust it to your needs.

* In `volumes:` section we can add all the folders we want, e.g. for using an external storage. They are generally mapped under the `/srv` or `/mnt` folders within the container. **In order to use those you will have to enable the "External Storage" plugin in Nextcloud.** It's installed by default, but disabled.

Assumes that it will be accessible from outside our local network. Therefore, the corresponding CNAME must be generated in our DNS provider.

Relies on **MariaDB** for its database. Although it can be configured with SQLite, having a dedicated database works better in the long run.

Also relies on **Redis** to maintain the in-memory cache. I recommend it even for single-user environments because the performance improvement is noticeable.

### *Environment Variables*

* `PUID` and `PGID` are the user and group IDs in numeric format (run `id` to find them)
* `TZ` is the time zone in `Continent/City` format. [List of zones](https://www.joda.org/joda-time/timezones.html)
* `DOCKERDIR` is the root directory containing all Docker services.
* `DOMAINNAME` is the name of our domain.
* `DB_PASSWORD` and `DB_ROOT_PASSWORD` are the passwords required for MariaDB. Picking two good, strong passwords is essential.

### *Before starting*

* As explained above, this service is dependent on the `proxy` network to be accessed from outside. Set `TRUSTED_PROXIES` in `docker-compose.yml`.

* Remember to create a CNAME record in the DNS provider. In `docker-compose.yml` it appears as `nextcloud.$DOMAINNAME`, although you can call it whatever you like.

* Create the `redis/` folder and the `sysctl.conf` file with its contents.
  
## Starting the container and initial configuration

```bash
docker compose up -d        → start Nextcloud in the background

docker logs nextcloud -f    → examine the logs to see if there are any problems (CTRL+c to exit)
```
</br>

In a browser, go to the address you have configured (`https://nextcloud.example.com`)

The configuration wizard will be displayed, so:

1. Enter the name of the user who will be the admin.
2. Enter the password.
3. (Optional) Install the recommended applications now.
4. Click on Storage and database.

</br>
  <p align="center" width="100%">
	<img width="33%" src="../.recursos/img/nextcloud/nextcloud_wizard_01.png">
  </p>
</br></br>

1. Here we select the database we will use. In this case, **MariaDB**
2. User: **`nextcloud`**
3. Password: **`DB_PASSWORD`** (set in `.env`)
4. Database name: **`nextcloud`**
5. Host: **`nextcloud-db:3306`**

  <p align="center" width="100%">
	<img width="33%" src="../.recursos/img/nextcloud/nextcloud_wizard_02.png">
  </p>
</br></br>

Log in with the credentials:

  <p align="center" width="100%">
	<img width="33%" src="../.recursos/img/nextcloud/nextcloud_login.png">
  </p>
</br>
<h3>
Done! Now we have a self-hosted personal cloud with access to dozens of applications.
</h3>