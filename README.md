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

 - Create instance in GCE by `docker-machine` and set up commands to the environment for the Docker client
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
~$ docker-machine rm <docker_instance_name>
```

## Homework 16

 - Create instance in GCE by `docker-machine`, test it and set up commands to the environment for the Docker client
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

 - Download the latest MongoDB image
```bash
~$ docker pull mongo:latest
```

 - Build images and test them
```bash
~$ cp ui/Dockerfile_1_0 ui/Dockerfile
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
dashishmakov/ui        3.0                 086371a13621        25 seconds ago       203MB
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

 - Restart running containers
```bash
~$ docker restart $(docker ps -q)
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
 ~$ docker-machine rm <docker_instance_name>
 ```