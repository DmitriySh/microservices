Microservices
=======


DevOps course, practices with [Google Cloud Platform](https://cloud.google.com/).

## Homework 15

Use [Docker](https://www.docker.com/) to create instance in GCE and publish docker image in [Docker Hub](https://hub.docker.com/)

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

 - At the end remove the docker machine and remote instance
```bash
~$ docker-machine kill <docker_instance_name>
~$ docker-machine rm <docker_instance_name>
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
~$ docker build -t dashishmakov/ui:1.0 -f ./ui/Dockerfile_1_0 ./ui/
~$ docker build -t dashishmakov/comment:1.0 ./comment/
~$ docker build -t dashishmakov/post:1.0 ./post-py/
~$ docker images -a | grep dashishmakov
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
~$ docker build -t dashishmakov/ui:2.0 -f ./ui/Dockerfile_2_0 ./ui/
~$ docker build -t dashishmakov/ui:3.0 -f ./ui/Dockerfile_3_0 ./ui/
~$ docker images dashishmakov/ui
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

 - At the end remove the docker machine and remote instance
```bash
~$ docker-machine kill <docker_instance_name>
~$ docker-machine rm <docker_instance_name>
```

## Homework 17


 - Create instance in GCE by `docker-machine` and change the environment variables for the `Docker Client`.
  It should connect to remote `Docker Engine`.
```bash
~$ docker-machine create --driver google \
   --google-project  docker-183019 \
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
~$ export USERNAME=dashishmakov
~$ cp .env.example .env
~$ docker-compose up -d
~$ docker-compose ps
          Name                       Command             State           Ports
---------------------------------------------------------------------------------------
microservices_comment_1    puma                          Up
microservices_mongo_db_1   docker-entrypoint.sh mongod   Up      27017/tcp
microservices_post_1       python3 post_app.py           Up
microservices_ui_1         puma                          Up      0.0.0.0:9292->9292/tcp
``` 

 - At the end remove docker containers and remote instance of docker machine
```bash
~$ docker-compose kill
~$ docker-compose rm
~$ docker-machine kill <docker_instance_name>
~$ docker-machine rm <docker_instance_name>
```
