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