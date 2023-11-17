# Docker Project README

I originally started the project with the help of the openVas9 Docker installation video linked in panopto, but ran into a problem with not being able to connet to localhost.

 I thought maybe the port was being used by something else so I used XAMPP to check, but openvas was the only one using it. 

 Also tried to flush my DNS, and checked my browser settings to make sure my domain security policy allowed localhost. Nothing worked.

 So I tried another way and came upon the greenbone community documentation:
`https://greenbone.github.io/docs/latest/22.4/container/index.html`

This was extremely helpful and is very detailed about every step. 

# Getting Started: Docker installation
To get started, first install WSL if you're using windows OS:
`https://learn.microsoft.com/en-us/windows/wsl/install`
 
 and install docker desktop:
https://docs.docker.com/desktop/install/windows-install/

Docker compose will then be installed as well automatically. 

# OpenVas/Greenbone installation

In WSL, first make sure the linux repositories are up to date

```shell
sudo apt update -y
```

Then, create a directory to store all the files for the greenbone docker environment, also set an environment variable for our destination directory to download and store all our files

```shell
export DOWNLOAD_DIR=$HOME/greenbone-dommunity-container && mkdir -p $DOWNLOAD_DIR
```

Next, we will run the Greenbone community edition with containers

```shell
cd $DOWNLOAD_DIR && curl -f -L https://greenbone.github.io/docs/latest/_static/docker-compose-22.4.yml -o docker-compose.yml
```

Then, run docker compose

```shell
docker compose -f $DOWNLOAD_DIR/docker-compose.yml -p greenbone-community-edition up -d
```
Like in the panopto video, a default user and password are both set to admin. So to change it:


```shell
docker compose -f $DOWNLOAD_DIR/docker-compose.yml -p greenbone-community-edition \
    exec -u gvmd gvmd gvmd --user=admin --new-password=<password>
```

![Alt text](<Screenshot 2023-11-17 001621.png>)


# docker-compose.yml code block:
```shell
services:
  vulnerability-tests:
    image: greenbone/vulnerability-tests
    environment:
      STORAGE_PATH: /var/lib/openvas/22.04/vt-data/nasl
    volumes:
      - vt_data_vol:/mnt

  notus-data:
    image: greenbone/notus-data
    volumes:
      - notus_data_vol:/mnt

  scap-data:
    image: greenbone/scap-data
    volumes:
      - scap_data_vol:/mnt

  cert-bund-data:
    image: greenbone/cert-bund-data
    volumes:
      - cert_data_vol:/mnt

  dfn-cert-data:
    image: greenbone/dfn-cert-data
    volumes:
      - cert_data_vol:/mnt
    depends_on:
      - cert-bund-data

  data-objects:
    image: greenbone/data-objects
    volumes:
      - data_objects_vol:/mnt

  report-formats:
    image: greenbone/report-formats
    volumes:
      - data_objects_vol:/mnt
    depends_on:
      - data-objects

  gpg-data:
    image: greenbone/gpg-data
    volumes:
      - gpg_data_vol:/mnt

  redis-server:
    image: greenbone/redis-server
    restart: on-failure
    volumes:
      - redis_socket_vol:/run/redis/

  pg-gvm:
    image: greenbone/pg-gvm:stable
    restart: on-failure
    volumes:
      - psql_data_vol:/var/lib/postgresql
      - psql_socket_vol:/var/run/postgresql

  gvmd:
    image: greenbone/gvmd:stable
    restart: on-failure
    volumes:
      - gvmd_data_vol:/var/lib/gvm
      - scap_data_vol:/var/lib/gvm/scap-data/
      - cert_data_vol:/var/lib/gvm/cert-data
      - data_objects_vol:/var/lib/gvm/data-objects/gvmd
      - vt_data_vol:/var/lib/openvas/plugins
      - psql_data_vol:/var/lib/postgresql
      - gvmd_socket_vol:/run/gvmd
      - ospd_openvas_socket_vol:/run/ospd
      - psql_socket_vol:/var/run/postgresql
    depends_on:
      pg-gvm:
        condition: service_started
      scap-data:
        condition: service_completed_successfully
      cert-bund-data:
        condition: service_completed_successfully
      dfn-cert-data:
        condition: service_completed_successfully
      data-objects:
        condition: service_completed_successfully
      report-formats:
        condition: service_completed_successfully

  gsa:
    image: greenbone/gsa:stable
    restart: on-failure
    ports:
      - 127.0.0.1:9392:80
    volumes:
      - gvmd_socket_vol:/run/gvmd
    depends_on:
      - gvmd

  ospd-openvas:
    image: greenbone/ospd-openvas:stable
    restart: on-failure
    hostname: ospd-openvas.local
    cap_add:
      - NET_ADMIN # for capturing packages in promiscuous mode
      - NET_RAW # for raw sockets e.g. used for the boreas alive detection
    security_opt:
      - seccomp=unconfined
      - apparmor=unconfined
    command:
      [
        "ospd-openvas",
        "-f",
        "--config",
        "/etc/gvm/ospd-openvas.conf",
        "--mqtt-broker-address",
        "mqtt-broker",
        "--notus-feed-dir",
        "/var/lib/notus/advisories",
        "-m",
        "666"
      ]
    volumes:
      - gpg_data_vol:/etc/openvas/gnupg
      - vt_data_vol:/var/lib/openvas/plugins
      - notus_data_vol:/var/lib/notus
      - ospd_openvas_socket_vol:/run/ospd
      - redis_socket_vol:/run/redis/
    depends_on:
      redis-server:
        condition: service_started
      gpg-data:
        condition: service_completed_successfully
      vulnerability-tests:
        condition: service_completed_successfully

  mqtt-broker:
    restart: on-failure
    image: greenbone/mqtt-broker
    networks:
      default:
        aliases:
          - mqtt-broker
          - broker

  notus-scanner:
    restart: on-failure
    image: greenbone/notus-scanner:stable
    volumes:
      - notus_data_vol:/var/lib/notus
      - gpg_data_vol:/etc/openvas/gnupg
    environment:
      NOTUS_SCANNER_MQTT_BROKER_ADDRESS: mqtt-broker
      NOTUS_SCANNER_PRODUCTS_DIRECTORY: /var/lib/notus/products
    depends_on:
      - mqtt-broker
      - gpg-data
      - vulnerability-tests

  gvm-tools:
    image: greenbone/gvm-tools
    volumes:
      - gvmd_socket_vol:/run/gvmd
      - ospd_openvas_socket_vol:/run/ospd
    depends_on:
      - gvmd
      - ospd-openvas

volumes:
  gpg_data_vol:
  scap_data_vol:
  cert_data_vol:
  data_objects_vol:
  gvmd_data_vol:
  psql_data_vol:
  vt_data_vol:
  notus_data_vol:
  psql_socket_vol:
  gvmd_socket_vol:
  ospd_openvas_socket_vol:
  redis_socket_vol:

```