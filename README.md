Microservices
=======


DevOps course, practices with [Google Cloud Platform](https://cloud.google.com/).

## Homework 15

[Docker](https://www.docker.com/) helps to package the software into standardized units for development, shipment and deployment. 
Containers are lightweight by design and ideal for enabling microservices application development. 

Use [Docker](https://www.docker.com/) to create instance in GCE and publish docker image in [Docker Hub](https://hub.docker.com/).

 - Configure `gcloud` and create new GCE project `docker`
```bash
~$ gcloud init
~$ gcloud auth application-default login
```

 - Create instance in GCE by `docker-machine` and change the environment variables for the `Docker Client`.
  It should connect to remote `Docker Engine`.
```bash
~$ docker-machine create --driver google \
--google-project <project_id> \
--google-zone europe-west1-b \
--google-machine-type g1-small \
--google-machine-image $(gcloud compute images list --filter ubuntu-1604-lts --uri) \
<docker_instance_name>

~$ eval $(docker-machine env <docker_instance_name>)
```

 - Display machines running Docker
```bash
~$ docker-machine ls
NAME                     ACTIVE   DRIVER   STATE     URL                        SWARM   DOCKER        ERRORS
<docker_instance_name>   *        google   Running   tcp://<host_ip>:2376           v17.09.0-ce
```

 - Build docker image
```bash
~$ docker build -t reddit:latest ./
~$ docker images -a
```

 - Run docker container
```bash
$ docker run --name reddit -d --network=host reddit:latest
```

 - Create firewall rule for reddit app
```bash
~$ gcloud compute firewall-rules create reddit-app \
--allow tcp:9292 --priority=65534 \
--target-tags=docker-machine \
--description="Allow TCP connections" \
--direction=INGRESS
```

 - Docker machine needs to have rules `docker-machines` and `reddit-app`
```bash
~$ gcloud compute firewall-rules list
NAME                    NETWORK  DIRECTION  PRIORITY  ALLOW                         DENY
default-allow-icmp      default  INGRESS    65534     icmp
default-allow-internal  default  INGRESS    65534     tcp:0-65535,udp:0-65535,icmp
default-allow-rdp       default  INGRESS    65534     tcp:3389
default-allow-ssh       default  INGRESS    65534     tcp:22
docker-machines         default  INGRESS    1000      tcp:2376
reddit-app              default  INGRESS    65534     tcp:9292
```

 - Open URL [http://\<host_ip\>:9292](http://\<host_ip\>:9292) and test the app
 - Connect to the remote instance by ssh
```bash
~$ docker-machine ssh <docker_instance_name>

Welcome to Ubuntu 16.04.3 LTS (GNU/Linux 4.10.0-37-generic x86_64)
 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
...
docker-user@docker-host:~$ lsb_release -d
Description:	Ubuntu 16.04.3 LTS
```

 - Auth and push docker image to [Docker Hub](https://hub.docker.com/)
```bash
~$ docker login
~$ docker tag reddit:latest <user-login>/otus-reddit:1.0
~$ docker push <user-login>/otus-reddit:1.0
The push refers to a repository [docker.io/dashishmakov/otus-reddit]
...
1.0: digest: sha256:6d60a3915efdd937a9ae0ad9e8c7b1fa2edc1a9d63000f9993cbcffa4fcca085 size: 3241
```

## Homework 16

 - Create instance in GCE by `docker-machine` and change the environment variables for the `Docker Client`.
  It should connect to remote `Docker Engine`.
```bash
~$ docker-machine create --driver google \
--google-project <project_id> \
--google-zone europe-west1-b \
--google-machine-type g1-small \
--google-machine-image $(gcloud compute images list --filter ubuntu-1604-lts --uri) \
<docker_instance_name>

~$ docker-machine ls
NAME          ACTIVE   DRIVER   STATE     URL                        SWARM   DOCKER        ERRORS
<docker_instance_name>   -        google   Running   tcp://<host_ip>:2376           v17.09.0-ce

~$ docker-machine ssh <docker_instance_name>
~$ eval $(docker-machine env <docker_instance_name>)
```

 - Download the latest [MongoDB](https://www.mongodb.com) image
```bash
~$ docker pull mongo:latest
```

 - Build images and test them
```bash
~compose$ docker build -t dashishmakov/ui:1.0 -f ./ui/Dockerfile_1_0 ./ui/
~compose$ docker build -t dashishmakov/comment:1.0 ./comment/
~compose$ docker build -t dashishmakov/post:1.0 ./post-py/
~compose$ docker images -a | grep dashishmakov
dashishmakov/ui        1.0                 0594004be8ef        3 minutes ago       779MB
dashishmakov/comment   1.0                 e18cb5c57fe5        4 minutes ago       774MB
dashishmakov/post      1.0                 1ab61dabb6fe        10 minutes ago      102MB
```

 - Create bridge-network for reddit app
```bash
~$ docker network create reddit
~$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
cac29504a2ac        bridge              bridge              local
cce3aff812f4        host                host                local
97112fc564cf        none                null                local
d85b1da7d878        reddit              bridge              local
```

 - Run containers, bind each of them to the network `reddit` and set network aliases
```bash
~$ docker run -d --network=reddit --network-alias=post_db --network-alias=comment_db mongo:latest
~$ docker run -d --network=reddit --network-alias=post dashishmakov/post:1.0
~$ docker run -d --network=reddit --network-alias=comment dashishmakov/comment:1.0
~$ docker run -d --network=reddit -p 9292:9292 dashishmakov/ui:1.0
~$ docker container ls
CONTAINER ID        IMAGE                      COMMAND                  CREATED             STATUS              PORTS                    NAMES
5ad6df21619c        dashishmakov/ui:1.0        "puma"                   27 seconds ago      Up 25 seconds       0.0.0.0:9292->9292/tcp   stoic_fermat
8e10a97ee1e6        dashishmakov/comment:1.0   "puma"                   37 seconds ago      Up 36 seconds                                pedantic_kalam
cc883b6c52ba        dashishmakov/post:1.0      "python3 post_app.py"    47 seconds ago      Up 46 seconds                                naughty_fermi
f8d620170918        mongo:latest               "docker-entrypoint..."   56 seconds ago      Up 55 seconds       27017/tcp                modest_blackwell
```

 - Open URL [http://<host_ip>:9292](http://<host_ip>:9292) and test the app
 - Stop running containers
```bash
~$ docker kill $(docker ps -q)
```

 - If you want to change the network aliases used for communication between applications
you should define new env values for each container
```bash
~$ docker run -d --network=reddit --network-alias=post_db --network-alias=net_comment_db \
 -e POST_SERVICE_HOST='net_post' \
 -e COMMENT_SERVICE_HOST='net_comment' \
 -e COMMENT_DATABASE_HOST='net_comment_db' \
mongo:latest

~$ docker run -d --network=reddit  --network-alias=net_post \
 -e POST_SERVICE_HOST='net_post' \
 -e COMMENT_SERVICE_HOST='net_comment' \
 -e COMMENT_DATABASE_HOST='net_comment_db' \
dashishmakov/post:1.0

~$ docker run -d --network=reddit --network-alias=net_comment \
 -e POST_SERVICE_HOST='net_post' \
 -e COMMENT_SERVICE_HOST='net_comment' \
 -e COMMENT_DATABASE_HOST='net_comment_db' dashishmakov/comment:1.0

~$ docker run -d --network=reddit \
 -p 9292:9292 \
 -e POST_SERVICE_HOST='net_post' \
 -e COMMENT_SERVICE_HOST='net_comment' \
 -e COMMENT_DATABASE_HOST='net_comment_db' \
dashishmakov/ui:1.0
```

 - Open URL [http://<host_ip>:9292](http://<host_ip>:9292) and test the app

 - Docker images are big and usually much larger than they need to be.
 True OS image runs either real hardware or virtualized hardware, but not inside a Docker container.
 Inside a container you don't want a full system. Use other docker files to build image 2.0 (ubuntu:16.04), image 3.0 (alpine:3.6)
 and test them with 1.0 (ruby:2.2)
```bash
~compose$ docker build -t dashishmakov/ui:2.0 -f ./ui/Dockerfile_2_0 ./ui/
~compose$ docker build -t dashishmakov/ui:3.0 -f ./ui/Dockerfile_3_0 ./ui/
~compose$ docker images dashishmakov/ui
dashishmakov/ui        3.0                 086371a13621        25 seconds ago       199MB
dashishmakov/ui        2.0                 caa68976405a        About an hour ago    454MB
dashishmakov/ui        1.0                 0594004be8ef        3 days ago           779MB
```

 - Stop running containers
```bash
~$ docker kill $(docker ps -q)
```

 - Create docker volume to save the data after restarting containers and mount it to the `mongo:latest`
```bash
~$ docker volume create reddit_db
~$ docker volume ls
~$ docker run -d --network=reddit --network-alias=post_db --network-alias=comment_db --mount source=reddit_db,target=/data/db mongo:latest
~$ docker run -d --network=reddit --network-alias=post dashishmakov/post:1.0
~$ docker run -d --network=reddit --network-alias=comment dashishmakov/comment:1.0
~$ docker run -d --network=reddit -p 9292:9292 dashishmakov/ui:1.0
```

 - Start new container with volume
```bash
~$ docker kill <mongo_container_id>
~$ docker run -d --network=reddit --network-alias=post_db --network-alias=comment_db --mount source=reddit_db,target=/data/db mongo:latest
~$ docker ps
CONTAINER ID        IMAGE                      COMMAND                  CREATED             STATUS              PORTS                    NAMES
d25e4f7d898d        dashishmakov/ui:1.0        "puma"                   2 hours ago         Up 15 seconds       0.0.0.0:9292->9292/tcp   friendly_yonath
1bdfd93a23cc        dashishmakov/comment:1.0   "puma"                   2 hours ago         Up 15 seconds                                agitated_jackson
76fbdb3c77e8        dashishmakov/post:1.0      "python3 post_app.py"    2 hours ago         Up 14 seconds                                mystifying_hypatia
3530edb57307        mongo:latest               "docker-entrypoint..."   2 hours ago         Up 13 seconds       27017/tcp                zen_hawking
```

 - Open URL [http://<host_ip>:9292](http://<host_ip>:9292), test the app and posts

## Homework 17


 - Create instance in GCE by `docker-machine` and change the environment variables for the `Docker Client`.
  It should connect to remote `Docker Engine`.
```bash
~$ docker-machine create --driver google \
   --google-project <project_id> \
   --google-zone europe-west1-b \
   --google-machine-type g1-small \
   --google-machine-image $(gcloud compute images list --filter ubuntu-1604-lts --uri) \
   docker-host

~$ docker-machine ls
NAME          ACTIVE   DRIVER   STATE     URL                        SWARM   DOCKER        ERRORS
docker-host   *        google   Running   tcp://<host_ip>:2376           v17.10.0-ce
```

 - Run container `net_test` with `none` network driver that has timer and will deleted after 100 sec. 
 Container has Loopback network interface only.
```bash
~$ docker run --network none --rm -d --name net_test joffotron/docker-net-tools -c "sleep 100"
~$ docker exec -ti net_test ifconfig
lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

 - Run container `net_test` with `host` network driver and test `ifconfig` outputs
```bash
~$ docker run --network host --rm -d --name net_test joffotron/docker-net-tools -c "sleep 100"
~$ docker exec -ti net_test ifconfig
~$ docker-machine ssh docker-host ifconfig
```

 - Create new `bridge` network (it is a default network in the docker and you no need to define key '--driver' explicitly, but just do it)
```bash
~$ docker network create reddit --driver bridge
~$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
e7a048591474        bridge              bridge              local
e5e41b5bb7b9        host                host                local
a9aca5eaa1f3        none                null                local
df49b0667c1c        reddit              bridge              local
```

 - Run containers, bind each of them to the network `reddit` and set network aliases because services search by DNS names from env variables
```bash
~$ docker run -d --network=reddit --network-alias=post_db --network-alias=comment_db mongo:latest
~$ docker run -d --network=reddit --network-alias=post dashishmakov/post:1.0
~$ docker run -d --network=reddit --network-alias=comment dashishmakov/comment:1.0
~$ docker run -d --network=reddit -p 9292:9292 dashishmakov/ui:1.0
```

- Open URL [http://<host_ip>:9292](http://<host_ip>:9292) and test the app

- Right now all services use one network interface.
Let's create different network interfaces and separate `ui` from `db`
```bash
~$ docker network create back_net --subnet=10.0.2.0/24
~$ docker network create front_net --subnet=10.0.1.0/24
~$ docker run -d --network=front_net -p 9292:9292 --name ui dashishmakov/ui:1.0
~$ docker run -d --network=back_net --name comment dashishmakov/comment:1.0
~$ docker run -d --network=back_net --name post dashishmakov/post:1.0
~$ docker run -d --network=back_net --name mongo_db --network-alias=post_db --network-alias=comment_db mongo:latest
~$ docker network connect front_net post
~$ docker network connect front_net comment
```

- Open URL [http://<host_ip>:9292](http://<host_ip>:9292) and test the app


- Let's look at the current Linux network stack on virtual instance in GCE
```bash
~$ docker-machine ssh docker-host
~$ sudo apt-get update && sudo apt-get install bridge-utils
~$ sudo docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
310a8bd102cf        back_net            bridge              local
e7a048591474        bridge              bridge              local
593663a63d49        front_net           bridge              local
e5e41b5bb7b9        host                host                local
a9aca5eaa1f3        none                null                local
df49b0667c1c        reddit              bridge              local

~$ ifconfig | grep br
br-310a8bd102cf Link encap:Ethernet  HWaddr 02:42:83:55:32:14
br-593663a63d49 Link encap:Ethernet  HWaddr 02:42:1d:7a:48:6c
br-df49b0667c1c Link encap:Ethernet  HWaddr 02:42:96:6c:be:1d

~$ brctl show br-593663a63d49
bridge name	bridge id		STP enabled	interfaces
br-593663a63d49		8000.02421d7a486c	no		veth3b6ae49
							vethc70235b
							vethcf4e66b
```

 - Show iptables
```bash
~$ sudo iptables -nL -t nat
...
Chain DOCKER (2 references)
target     prot opt source               destination
RETURN     all  --  0.0.0.0/0            0.0.0.0/0
RETURN     all  --  0.0.0.0/0            0.0.0.0/0
RETURN     all  --  0.0.0.0/0            0.0.0.0/0
RETURN     all  --  0.0.0.0/0            0.0.0.0/0
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:9292 to:10.0.1.2:9292
```

 - Find the `docker-proxy` process listening on TCP port 9292
```bash
~$ ps ax | grep docker-proxy
```

 - Stop and remove running containers
```bash
~$ docker kill $(docker ps -q)
~$ docker rm $(docker ps -aq)
```

#### docker-compose

A service definition contains configuration which will be applied to each container started for that service, 
much like passing command-line parameters to `docker run`, `docker network`, `docker volume`

 - Set environment variables for `docker-compose`
```bash
~compose$ export USERNAME=dashishmakov
~compose$ cp .env.example .env
~compose$ docker-compose up -d
~compose$ docker-compose ps
          Name                       Command             State           Ports
---------------------------------------------------------------------------------------
microservices_comment_1    puma                          Up
microservices_mongo_db_1   docker-entrypoint.sh mongod   Up      27017/tcp
microservices_post_1       python3 post_app.py           Up
microservices_ui_1         puma                          Up      0.0.0.0:9292->9292/tcp
```

## Homework 20, 21

1.1) [Prometheus](https://prometheus.io) is a powerful time-series monitoring service, providing a flexible platform for
monitoring software products. Let's run and get acquainted with this product.

 - create instance in GCE by `docker-machine`. Change the environment variables for the `Docker Client` and connect to the remote `Docker Engine`
```bash
$ docker-machine create --driver google \
--google-project <project_id> \
--google-machine-image https://www.googleapis.com/compute/v1/projects/ubuntu-os-cloud/global/images/family/ubuntu-1604-lts \
--google-machine-type n1-standard-1 \
--google-zone europe-west1-b \
vm1

$ eval $(docker-machine env vm1)
```

 - check firewall-rules:
```bash
$ gcloud compute firewall-rules list
NAME                    NETWORK  DIRECTION  PRIORITY  ALLOW                         DENY
default-allow-icmp      default  INGRESS    65534     icmp
default-allow-internal  default  INGRESS    65534     tcp:0-65535,udp:0-65535,icmp
default-allow-rdp       default  INGRESS    65534     tcp:3389
default-allow-ssh       default  INGRESS    65534     tcp:22
docker-machines         default  INGRESS    1000      tcp:2376
prometheus-default      default  INGRESS    1000      tcp:9090
puma-default            default  INGRESS    1000      tcp:9292
```

 - create if you do not have one of them
```bash
~$ gcloud compute firewall-rules create default-allow-ssh --allow tcp:22 --priority=65534 --description="Allow SSH connections" --direction=INGRESS
~$ gcloud compute firewall-rules create prometheus-default --allow tcp:9090
~$ gcloud compute firewall-rules create puma-default --allow tcp:9292
```

 - run [Prometheus](https://prometheus.io) from the DockerHub image
```bash
~compose$ docker run --rm -p 9090:9090 -d --name prometheus prom/prometheus
~compose$ docker ps
```

 - open URL [http://<host_ip>:9090/targets](http://<host_ip>:9090/targets) and `[http://<host_ip>:9090/metrics](http://<host_ip>:9090/metrics)

 - stop docker instance [Prometheus](https://prometheus.io)
```bash
~compose$ docker stop prometheus
```

1.2) [Prometheus](https://prometheus.io) will monitor all microservices, so we need a container with Prometheus that could communicate
 over the network with all other services and config it in `docker-compose` file

 - set environment variables for `docker-compose`
```bash
~compose$ export USERNAME=<dockerhub_login>
~compose$ cp .env.example .env
```

 - build custom images with [Prometheus](https://prometheus.io) and other microservices
```bash
~compose/prometheus$ bash docker_build.sh
~compose/ui$ bash docker_build.sh
~compose/post-py$ bash docker_build.sh
~compose/comment$ bash docker_build.sh
$ docker images
REPOSITORY                      TAG                 IMAGE ID            CREATED             SIZE
dashishmakov/ui                 latest              dfac89a27201        43 minutes ago      204MB
dashishmakov/comment            latest              ce03118d56db        2 hours ago         778MB
dashishmakov/post               latest              8cafeba72426        2 hours ago         101MB
dashishmakov/prometheus         latest              1c9abba20bb2        2 hours ago         80.2MB
```

 - run all containers
```bash
~compose$ docker-compose up -d
~compose$ docker-compose ps
           Name                         Command               State           Ports
--------------------------------------------------------------------------------------------
microservices_comment_1      puma                             Up
microservices_mongo_db_1     docker-entrypoint.sh mongod      Up      27017/tcp
microservices_post_1         python3 post_app.py              Up      5000/tcp
microservices_prometheus_1   /bin/prometheus -config.fi ...   Up      0.0.0.0:9090->9090/tcp
microservices_ui_1           puma                             Up      0.0.0.0:9292->9292/tcp
```

 - open URL [http://<host_ip>:9292/targets](http://<host_ip>:9292/targets), [http://<host_ip>:9090/metrics](http://<host_ip>:9090/metrics),
 [http://<host_ip>:9090/targets](http://<host_ip>:9090/targets) and test the app
 - `<host_ip>` is an external host ip: `docker-machine ip vm1`

1.3) `Healthcheck` is a part of metrics for each service and runs into each of them (part of source code).
Microservices are dependent from each other and `healthcheck` indicates availability all endpoints for concrete service (1 = healthy, 0 = unhealthy)

 - open URL [http://<host_ip>:9090] and find graph `ui_health`
 - stop `post` service and refresh graph
```bash
~compose$ docker-compose stop post
```

 - open graphs `ui_health_post_availability` and `ui_health_comment_availability` and test each of them
 - start `post` service should fix `Healthcheck`
```bash
~compose$ docker-compose start post
```

1.4) [Node exporter](https://github.com/prometheus/node_exporter) is a part of project [Prometheus](https://prometheus.io). It helps to collect hardware and *NIX kernels metrics of host machine (main virtual machine) for [Prometheus](https://prometheus.io);
[MongoDB exporter](https://github.com/percona/mongodb_exporter) collects metrics about sharding, replication and storage engines

 - rebuild [Prometheus](https://prometheus.io) image if previous version don't have access to [Node exporter](https://github.com/prometheus/node_exporter), build [MongoDB exporter](https://github.com/percona/mongodb_exporter) image and restart microservices
```bash
 ~compose/prometheus$ bash docker_build.sh
 ~compose/mongodb-exporter$ bash docker_build.sh
 ~compose$ docker-compose down
 ~compose$ docker-compose up -d
 ~compose$ docker-compose ps
              Name                            Command               State           Ports
--------------------------------------------------------------------------------------------------
microservices_comment_1            puma                             Up
microservices_mongo_db_1           docker-entrypoint.sh mongod      Up      27017/tcp
microservices_mongodb-exporter_1   mongodb_exporter                 Up      9001/tcp
microservices_node-exporter_1      /bin/node_exporter --path. ...   Up      9100/tcp
microservices_post_1               python3 post_app.py              Up      5000/tcp
microservices_prometheus_1         /bin/prometheus --config.f ...   Up      0.0.0.0:9090->9090/tcp
microservices_ui_1                 puma                             Up      0.0.0.0:9292->9292/tcp
```

 - open URL [http://<host_ip>:9090/targets] and test `node_cpu` metric
 - push all microservice images to [Docker Hub](https://hub.docker.com) repository
```bash
~compose$ docker login
~compose$ docker push $USERNAME/ui
~compose$ docker push $USERNAME/post
~compose$ docker push $USERNAME/comment
~compose$ docker push $USERNAME/prometheus
~compose$ docker push $USERNAME/mongodb_exporter
```

## Homework 23

1.1) Docker's container monitoring. [cAdvisor](https://github.com/google/cadvisor) analyzes resource usage and performance characteristics of running containers. 
It collects, aggregates, processes, and might to exports information to various storages such as 
[Prometheus](https://prometheus.io), [ElasticSearch](https://www.elastic.co), [InfluxDB](https://www.influxdata.com), [Kafka](http://kafka.apache.org) and simple stdout.

 - create instance in GCE by `docker-machine`. Change the environment variables for the `Docker Client` and connect to the remote `Docker Engine`
```bash
$ docker-machine create --driver google \
--google-project <project_id> \
--google-machine-image https://www.googleapis.com/compute/v1/projects/ubuntu-os-cloud/global/images/family/ubuntu-1604-lts \
--google-machine-type n1-standard-1 \
--google-zone europe-west1-b \
--google-open-port 80/tcp \
--google-open-port 3000/tcp \
--google-open-port 8080/tcp \
--google-open-port 9090/tcp \
--google-open-port 9292/tcp \
vm1

$ eval $(docker-machine env vm1)
```

 - new service `cadvisor` should be defined in `docker-compose` file and [Prometheus](https://prometheus.io) should get information from here
 - set environment variables for docker-compose
```bash
~compose$ export USERNAME=<dockerhub_login>
~compose$ cp .env.example .env
```

  - rebuild [Prometheus](https://prometheus.io) image and start microservices
```bash
~compose/prometheus$ bash docker_build.sh
~compose$ docker-compose up -d
~compose$ docker-compose ps
              Name                            Command               State           Ports
--------------------------------------------------------------------------------------------------
microservices_cadvisor_1           /usr/bin/cadvisor -logtostderr   Up      0.0.0.0:8080->8080/tcp
microservices_comment_1            puma                             Up
microservices_mongo_db_1           docker-entrypoint.sh mongod      Up      27017/tcp
microservices_mongodb-exporter_1   mongodb_exporter                 Up      9001/tcp
microservices_node-exporter_1      /bin/node_exporter --path. ...   Up      9100/tcp
microservices_post_1               python3 post_app.py              Up      5000/tcp
microservices_prometheus_1         /bin/prometheus --config.f ...   Up      0.0.0.0:9090->9090/tcp
microservices_ui_1                 puma                             Up      0.0.0.0:9292->9292/tcp
```

 - open URL [http://<host_ip>:8080/docker] and watch detailed information for each one; [http://<host_ip>:8080/metrics] defines metrics for Prometheus, `container_memory_usage_bytes` for example
 - open URL [http://<host_ip>:9090] and test metric `container_memory_usage_bytes` and others in [Prometheus](https://prometheus.io)

1.2.1) Use [Grafana](https://grafana.com) for visualizing metrics data from docker containers

 - new service `grafana` should be defined in `docker-compose` file
 - start service  `grafana`
```bash
~compose$ docker-compose up -d grafana
~compose$ docker-compose ps
              Name                            Command               State           Ports
--------------------------------------------------------------------------------------------------
microservices_cadvisor_1           /usr/bin/cadvisor -logtostderr   Up      0.0.0.0:8080->8080/tcp
microservices_comment_1            puma                             Up
microservices_grafana_1            /run.sh                          Up      0.0.0.0:3000->3000/tcp
microservices_mongo_db_1           docker-entrypoint.sh mongod      Up      27017/tcp
microservices_mongodb-exporter_1   mongodb_exporter                 Up      9001/tcp
microservices_node-exporter_1      /bin/node_exporter --path. ...   Up      9100/tcp
microservices_post_1               python3 post_app.py              Up      5000/tcp
microservices_prometheus_1         /bin/prometheus --config.f ...   Up      0.0.0.0:9090->9090/tcp
microservices_ui_1                 puma                             Up      0.0.0.0:9292->9292/tcp
```

 - open URL [http://<host_ip>:3000/docker], login and add new data source, name: `Prometheus Server`, type: `Prometheus`, url: `http://<host_ip>:9090`
 - download shared dashboard `Docker and system monitoring` from [Dashboards](https://grafana.com/dashboards) and import it

1.2.2) Let's create new dashboard `UI Service Monitoring` for [Grafana](https://grafana.com):
 
 - `main menu` -> `dashboards` -> `new`
   - `graph` -> `edit panel title` -> `metrics` tab -> `data source`: Prometheus Server -> select metric `ui_request_count`
   - tune graph: refresh `every 10 seconds` and show `last 15 minutes`
   - `general` tab -> `Title`: `UI HTTP Requests`
 - save template with name: `Add UI HTTP Requests`

 - add new row `Graph` for our custom dashboard
   - `graph` -> `edit panel title` -> `metrics` tab -> `data source`: Prometheus Server -> select metric pattern `rate(ui_request_count{http_status=~"^[45].*"}[5m])`
   - tune graph: refresh `every 10 seconds` and show `last 15 minutes`
   - `general` tab -> `Title`: `Rate of UI HTTP Requests with Error`
   - for test only open URL `http://<host_ip>:9292/nonexistent` and register exception with undefined endpoint
 - save template with name: `Add HTTP Requests with Error status code`

 - add 3rd row `Graph` for our custom dashboard
   - `graph` -> `edit panel title` -> `metrics` tab -> `data source`: Prometheus Server -> select metric pattern `histogram_quantile(0.95, sum(rate(ui_request_latency_seconds_bucket[5m])) by (le))`
   - `general` tab -> `Title`: `HTTP response time 95th percentile`
 - save template with name: `Add HTTP response time 95th percentile`

 - each dashboard has own `version history` of changes, look at configuration
 - export dashbord to JSON file with name: `UI Service Monitoring`

1.2.3) Let's create new one dashboard `Business Logic Monitoring`:

 - title: `Post count`, metrics query: `rate(post_count[1h])`, save template
 - title: `Comment Count`, metrics query: `rate(comment_count[1h])`, save template
 - export dashbord to JSON file


 1.3) Alerts help to know about some problems in production environment at the same time when that happens. 
 Let's add WebHook for your own [Slack](https://slack.com) channel to get notifications in some accident cases. 
 [Alertmanager](https://prometheus.io/docs/alerting/alertmanager/) is a component for [Prometheus](https://prometheus.io) that handle alerts and rote sufisticated notifications

 - new service `alertmanager` should be defined in `docker-compose.yml` and `prometheus.yml` that [Prometheus](https://prometheus.io) should to know about it 
 - edit `config.yml` and define your own `slack_api_url` and `receivers.slack_configs.channel`
 - build image of `alertmanager` and run container
```bash
~compose/alertmanager$ ./docker_build.sh
~compose$ docker-compose up -d alertmanager
```
 - if you edit `alert.rules.yml` you should rebuild `prometheus` docker image and run it again
```bash
~compose$ docker-compose stop prometheus
~compose$ docker-compose rm prometheus
~compose$ docker-compose up -d prometheus
```
 - open URL [http://<host_ip>:9090] -> set `up` -> submit `execute` and look for this graph
 - stop container `post` and wait 1 minute, [Slack](https://slack.com) should receive notification `[FIRING:1] InstanceDown (post:5000 post page)`
```bash
~compose$ docker-compose stop post
... wait 1m
~compose$ docker-compose up -d post
```



---

At the end remove docker containers and remote instance of docker machine
```bash
~compose$ docker-compose kill
~compose$ docker-compose rm
~compose$ docker-machine kill <docker_instance_name>
~compose$ docker-machine rm <docker_instance_name>
```

## Homework 27

1.1) [Docker](https://www.docker.com/) include `swarm` mode for natively managing a cluster of Docker Engines called a [Docker Swarm](https://docs.docker.com/engine/swarm/).
[Docker Swarm](https://docs.docker.com/engine/swarm/) available out of the box for cluster management and orchestration features.

 - create several instances in GCE by `docker-machine`. 
```bash
~swarm$ docker-machine create --driver google \
   --google-project <project_id> \
   --google-zone europe-west1-b \
   --google-machine-type g1-small \
   --google-machine-image $(gcloud compute images list --filter ubuntu-1604-lts --uri) \
   master-1
~swarm$ docker-machine create --driver google \
   --google-project <project_id> \
   --google-zone europe-west1-b \
   --google-machine-type g1-small \
   --google-machine-image $(gcloud compute images list --filter ubuntu-1604-lts --uri) \
   worker-1
~swarm$ docker-machine create --driver google \
   --google-project <project_id> \
   --google-zone europe-west1-b \
   --google-machine-type g1-small \
   --google-machine-image $(gcloud compute images list --filter ubuntu-1604-lts --uri) \
   worker-2
```

 - Change the environment variables for the `Docker Client` and connect to the remote `Docker Engine` (node `master-1`)
```bash
~swarm$ eval $(docker-machine env master-1)
~swarm$ export USERNAME=dashishmakov
```

 - rebuild docker image for `ui` service and push latest version to [Docker Hub](https://hub.docker.com) repository
```bash
~swarm/ui$ bash docker_build.sh
~swarm$ docker push $USERNAME/ui
```

 - init [Docker Swarm](https://docs.docker.com/engine/swarm/)
   - node `master-1` switch to `swarm` mode
   - node to become `swarm` manager node
   - node assigned a hostname
   - node listen port 2377
   - node has status `active` and could get task from scheduler
   - init distributed persistent key-value storage for orchestration
   - generates self-signed root certificate for `swarm`
   - generates tokens to bind `Worker` и `Manager` nodes to the cluster
   - creates overlay-network `Ingress` to publish ports outside the swarm. All nodes participate in an ingress routing mesh
```bash
~swarm$ docker swarm init --advertise-addr <master-1_internal_ip>:2377
Swarm initialized: current node (1scjwp239x4ajg5kabr1kv0wm) is now a manager.
To add a worker to this swarm, run the following command:
    docker swarm join --token SWMTKN-1-0wo27ownjz1zgw0zsbit5e7aa69r7td8a91tpvnu5m32cenbtp-bmrtyoioou9zupoc2w9sug6zv <master-1_internal_ip>:2377
To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

 - add `worker-1` and `worker-2` to `swarm` cluster
```bash
~swarm$ docker-machine ssh worker-1
Welcome to Ubuntu 16.04.3 LTS (GNU/Linux 4.10.0-40-generic x86_64)
...
docker-user@worker-1:~$ sudo docker swarm join \
--token SWMTKN-1-0wo27ownjz1zgw0zsbit5e7aa69r7td8a91tpvnu5m32cenbtp-bmrtyoioou9zupoc2w9sug6zv \
<master-1_internal_ip>:2377
This node joined a swarm as a worker.

~swarm$ docker-machine ssh worker-2
Welcome to Ubuntu 16.04.3 LTS (GNU/Linux 4.10.0-40-generic x86_64)
...
docker-user@worker-2:~$ sudo docker swarm join \
--token SWMTKN-1-0wo27ownjz1zgw0zsbit5e7aa69r7td8a91tpvnu5m32cenbtp-bmrtyoioou9zupoc2w9sug6zv \
<master-1_internal_ip>:2377
This node joined a swarm as a worker.
```

 - print all nodes in the `swarm`
```bash
~swarm$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
1scjwp239x4ajg5kabr1kv0wm *   master-1            Ready               Active              Leader
h4rf9on49q2v2n623ei0a2zlg     worker-1            Ready               Active
whc2xwhg3rfvbzxp3icpirnd9     worker-2            Ready               Active
```

1.2) [Docker Compose](https://docs.docker.com/compose) is a good tool for defining and running multi-container of Docker applications.
 [Docker Compose](https://docs.docker.com/compose) is the heart of [Docker Stack](https://docs.docker.com/engine/reference/commandline/stack/) 
 and define stack of services for `swarm`
 
 - let's deploy a new [Docker Stack](https://docs.docker.com/engine/reference/commandline/stack/)
```bash
~swarm$ docker stack deploy --compose-file=<(docker-compose -f docker-compose.yml config 2>/dev/null) DEV
~swarm$ docker stack services DEV
ID                  NAME                MODE                REPLICAS            IMAGE                         PORTS
kuk8aiu1zb9h        DEV_ui              replicated          1/1                 dashishmakov/ui:latest        *:9292->9292/tcp
rhq65y0tlqrf        DEV_mongo           replicated          1/1                 mongo:3.2
td9jgtmq6buy        DEV_comment         replicated          1/1                 dashishmakov/comment:latest
vlykfrg53f15        DEV_post            replicated          1/1                 dashishmakov/post:latest
```

 - add `label` to the master node
```bash
~swarm$ docker node update --label-add reliability=high master-1
master-1
```

 - describe all available `labels`
```bash
~swarm$ docker node ls -q | xargs docker node inspect  -f '{{ .ID }} [{{ .Description.Hostname }}]: {{ .Spec.Labels }}'
```

 - update [Docker Stack](https://docs.docker.com/engine/reference/commandline/stack/)
```bash
~swarm$ docker stack deploy --compose-file=<(docker-compose -f docker-compose.yml config 2>/dev/null) DEV
Updating service DEV_mongo (id: rhq65y0tlqrfb5fwb110xrsts)
Updating service DEV_post (id: vlykfrg53f15t41di63ocpefo)
Updating service DEV_ui (id: kuk8aiu1zb9hxxwhrvbwjy0o0)
Updating service DEV_comment (id: td9jgtmq6buyxj7897s2sa6n4)
```

 - check statuses of containers and scaling by all nodes
   - 1x scale on master node (mongo) - no scaling
   - 2x scale on worker nodes (ui, pos, comment) - replicated scaling
   - 3x scale on master and worker nodes (node-exporter) - global replicated
```bash
~swarm$ docker stack ps DEV
ID                  NAME                                          IMAGE                         NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
mit4pxws64uk        DEV_node-exporter.h4rf9on49q2v2n623ei0a2zlg   prom/node-exporter:v0.15.0    worker-1            Running             Running 25 seconds ago
bqn4962cey46        DEV_node-exporter.1scjwp239x4ajg5kabr1kv0wm   prom/node-exporter:v0.15.0    master-1            Running             Running 25 seconds ago
zhpe6t4zw4qa        DEV_node-exporter.whc2xwhg3rfvbzxp3icpirnd9   prom/node-exporter:v0.15.0    worker-2            Running             Running 23 seconds ago
oqsmddkcj1iq        DEV_comment.1                                 dashishmakov/comment:latest   worker-2            Running             Running 12 hours ago
u699vk83e2i7        DEV_ui.1                                      dashishmakov/ui:latest        worker-2            Running             Running 12 hours ago
jic5u0vk2mkw        DEV_post.1                                    dashishmakov/post:latest      worker-1            Running             Running 12 hours ago
quvmp86hzb8c        DEV_mongo.1                                   mongo:3.2                     master-1            Running             Running 12 hours ago
bivw1mfilrc0        DEV_comment.2                                 dashishmakov/comment:latest   worker-1            Running             Running 23 minutes ago
fgucvzrvfbdo        DEV_ui.2                                      dashishmakov/ui:latest        worker-1            Running             Running 23 minutes ago
iwgd5gjlxfmk        DEV_post.2                                    dashishmakov/post:latest      worker-2            Running             Running 24 minutes ago
```

- add new one worker node to `swarm`
```bash
~swarm$ docker-machine create --driver google \
   --google-project <project_id> \
   --google-zone europe-west1-b \
   --google-machine-type g1-small \
   --google-machine-image $(gcloud compute images list --filter ubuntu-1604-lts --uri) \
   worker-3
   
~swarm$ docker swarm join-token worker
To add a worker to this swarm, run the following command:
    docker swarm join --token SWMTKN-1-0wo27ownjz1zgw0zsbit5e7aa69r7td8a91tpvnu5m32cenbtp-bmrtyoioou9zupoc2w9sug6zv <master-1_internal_ip>:2377

~swarm$ docker-machine ssh worker-3
Welcome to Ubuntu 16.04.3 LTS (GNU/Linux 4.10.0-40-generic x86_64)
...
docker-user@worker-3:~$ sudo docker swarm join \
--token SWMTKN-1-0wo27ownjz1zgw0zsbit5e7aa69r7td8a91tpvnu5m32cenbtp-bmrtyoioou9zupoc2w9sug6zv \
<master-1_internal_ip>:2377
This node joined a swarm as a worker.
```

 - check statuses of containers and scaling by all 4 nodes
```bash
~swarm$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
1scjwp239x4ajg5kabr1kv0wm *   master-1            Ready               Active              Leader
h4rf9on49q2v2n623ei0a2zlg     worker-1            Ready               Active
whc2xwhg3rfvbzxp3icpirnd9     worker-2            Ready               Active
pt2fhi36b036eo2yvbm8giplk     worker-3            Ready               Active

~swarm$ docker stack ps DEV
 ID                  NAME                                          IMAGE                         NODE                DESIRED STATE       CURRENT STATE               ERROR               PORTS
 vyjf8tvz9f3n        DEV_node-exporter.pt2fhi36b036eo2yvbm8giplk   prom/node-exporter:v0.15.0    worker-3            Running             Running 10 minutes ago
 mit4pxws64uk        DEV_node-exporter.h4rf9on49q2v2n623ei0a2zlg   prom/node-exporter:v0.15.0    worker-1            Running             Running 26 minutes ago
 bqn4962cey46        DEV_node-exporter.1scjwp239x4ajg5kabr1kv0wm   prom/node-exporter:v0.15.0    master-1            Running             Running 26 minutes ago
 zhpe6t4zw4qa        DEV_node-exporter.whc2xwhg3rfvbzxp3icpirnd9   prom/node-exporter:v0.15.0    worker-2            Running             Running 26 minutes ago
 oqsmddkcj1iq        DEV_comment.1                                 dashishmakov/comment:latest   worker-2            Running             Running 12 hours ago
 u699vk83e2i7        DEV_ui.1                                      dashishmakov/ui:latest        worker-2            Running             Running 12 hours ago
 jic5u0vk2mkw        DEV_post.1                                    dashishmakov/post:latest      worker-1            Running             Running 12 hours ago
 quvmp86hzb8c        DEV_mongo.1                                   mongo:3.2                     master-1            Running             Running 12 hours ago
 bivw1mfilrc0        DEV_comment.2                                 dashishmakov/comment:latest   worker-1            Running             Running about an hour ago
 fgucvzrvfbdo        DEV_ui.2                                      dashishmakov/ui:latest        worker-1            Running             Running about an hour ago
 iwgd5gjlxfmk        DEV_post.2                                    dashishmakov/post:latest      worker-2            Running             Running about an hour ago
```

 - increase min replica values and check again
 
 1.3) [Docker Swarm](https://docs.docker.com/engine/swarm/) has routing mesh, load balancer and decentralization architecture. 
 It helps to route all requests between worker nodes in cluster. Do several requests on concrete host in example
 
```bash
~swarm$ docker-machine ip $(docker-machine ls -q)
<external_ip_master-1>
<external_ip_worker-1>
<external_ip_worker-2>
<external_ip_worker-3>
~swarm$ curl <external_ip_worker-3>:9292 | grep "Microservices Reddit in"
<a class='navbar-brand' href='/'>Microservices Reddit in DEV 03a8760bef10 container</a>
...
<a class='navbar-brand' href='/'>Microservices Reddit in DEV 3e1e5763d96e container</a>
...
<a class='navbar-brand' href='/'>Microservices Reddit in DEV 5d1bf5267ca2 container</a>
```

1.4) Keep all services into the file `docker-compose.yml` and all infrastructure services to the file `docker-compose.infra.yml` 

```bash
~swarm$ docker stack rm DEV
~swarm$ docker stack deploy --compose-file=<(docker-compose -f docker-compose.infra.yml -f docker-compose.yml config 2>/dev/null) DEV
```



---

At the end remove docker containers and remote instance of docker machine
```bash
~swarm$ docker stack rm DEV
~swarm$ docker-machine kill $(docker-machine ls -q)
~swarm$ docker-machine rm $(docker-machine ls -q)
```
