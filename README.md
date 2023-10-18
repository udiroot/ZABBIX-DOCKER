# ZABBIX-DOCKER

-- INSTALAÇÃO DO ORACLE LINUX 9
https://oracle-base.com/articles/linux/oracle-linux-9-installation

OBS: Durante a instalação eu criei um volume no linux chamado /dockervolume

-- INSTALAÇÃO O DOCKER EM SISTEMAS LINUX BASEADO EM OL9 (Oracle Linux 9)

dnf install -y dnf-utils zip unzip
dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
dnf remove -y runc
dnf install -y docker-ce --nobest
systemctl enable docker.service
systemctl start docker.service
systemctl status docker.service
docker info
docker version

-- INSTALAÇÃO DOS CONTAINER EM DOCKER

#CRIANDO A REDE PARA COMUNICAÇÃO
docker network create --subnet 172.20.0.0/16 --ip-range 172.20.240.0/20 zabbix-net

#CRIANDO O CONTAINER DE BANCO DE DADOS
mkdir /dockervolume/data/

Limpar o volume do filesystem: rm -rf /dockervolume/* ou rm -rf /dockervolume/data/*

docker run --name some-postgres-server -t -e POSTGRES_USER="zabbix" -e POSTGRES_PASSWORD="zabbix" -e POSTGRES_DB="zabbixdb" -p 5432:5432 -v /dockervolume/postgresql-data:/var/lib/postgresql/data --network=zabbix-net --restart unless-stopped -d postgres:latest

#CRIANDO O CONTAINER DO ZABBIX SERVER
mkdir /dockervolume/zbx-server

docker run --name zabbix-server-pgsql -t -e DB_SERVER_HOST="some-postgres-server" -e POSTGRES_USER="zabbix" -e POSTGRES_PASSWORD="zabbix" -e POSTGRES_DB="zabbixdb" -e  ZBX_ALLOWUNSUPPORTEDDBVERSIONS=1 -e ZBX_STARTPOLLERS=10 -e ZBX_STARTPOLLERSUNREACHABLE=2 -e ZBX_STARTPINGERS=5 --network=zabbix-net -p 10051:10051 -v /dockervolume/zbx-server:/usr/lib/zabbix/alertscripts -v /dockervolume/zbx-server:/usr/lib/zabbix/externalscripts -v /dockervolume/zbx-server:/var/lib/zabbix/mibs --restart unless-stopped -d zabbix/zabbix-server-pgsql:alpine-6.0-latest

#CRIANDO O CONTAINER DO FRONTEND DO ZABBIX
mkdir /dockervolume/zbx-web

docker run --name zabbix-web-nginx-pgsql --cpus 15.0 --memory 512m -t -e ZBX_SERVER_HOST="zabbix-server-pgsql" -e DB_SERVER_HOST="some-postgres-server" -e POSTGRES_USER="zabbix" -e  POSTGRES_PASSWORD="zabbix" -e POSTGRES_DB="zabbixdb" --network=zabbix-net -p 443:8443 -p 80:8080 -v /dockervolume/zbx-web:/etc/ssl/nginx:ro --restart unless-stopped -d  zabbix/zabbix-web-nginx-pgsql:alpine-6.0-latest

#CRIANDO UM CONTAINER DE AGENT ZABBIX
obs: caso o container não suba, suba ele sem o volume, pegue o conf do agente e cole em um docker volume, depois rode o comando abaixo com o volume já montado

obs1: para que o agente do zabbix funcione, precisa ver qual IP do agente do zabbix pegou pela rede docker network inspect zabbix-net e add ele no host zabbix server cadastrado no frontend.

mkdir /dockervolume/zbx-agent

docker run --name some-zabbix-agent --cpus 1.0 --memory 512m -e ZBX_HOSTNAME="urootserver" --network=zabbix-net --link zabbix-server-pgsql:zabbix-server -v /dockervolume/zbx-agent:/etc/zabbix --init -p 10050:10050 --privileged --restart unless-stopped -d zabbix/zabbix-agent:alpine-6.0-latest
