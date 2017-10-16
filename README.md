Microservices
=======


DevOps course, practices with [Google Cloud Platform](https://cloud.google.com/).


Make `gcloud` default auth
~$ gcloud auth application-default login


Create instance in GCE by docker
~$ docker-machine create --driver google \
--google-project docker-181710 \
--google-zone europe-west1-b \
--google-machine-type f1-micro \
--google-machine-image $(gcloud compute images list --filter ubuntu-1604-lts --uri) \
<docker_instance_name>


Check machines running Docker
~$ docker-machine ls
NAME                     ACTIVE   DRIVER   STATE     URL                        SWARM   DOCKER        ERRORS
<docker_instance_name>   *        google   Running   tcp://35.195.32.106:2376           v17.09.0-ce


Build docker image
~$ eval $(docker-machine env docker-host)
~$ docker build -t reddit:latest .
~$ docker images -a


Run docker container
$ docker run --name reddit -d --network=host reddit:latest


Create firewall rule for reddit app
~$ gcloud compute firewall-rules create reddit-app \
--allow tcp:9292 --priority=65534 \
--target-tags=docker-machine \
--description="Allow TCP connections" \
--direction=INGRESS


Docker machine needs to have rules `docker-machines` and `reddit-app`
~$ gcloud compute firewall-rules list
NAME                    NETWORK  DIRECTION  PRIORITY  ALLOW                         DENY
default-allow-icmp      default  INGRESS    65534     icmp
default-allow-internal  default  INGRESS    65534     tcp:0-65535,udp:0-65535,icmp
default-allow-rdp       default  INGRESS    65534     tcp:3389
default-allow-ssh       default  INGRESS    65534     tcp:22
docker-machines         default  INGRESS    1000      tcp:2376
reddit-app              default  INGRESS    65534     tcp:9292


Open URL http://35.195.32.106:9292


Auth and push docker image to Docker Hub
~$ docker login
~$ docker tag reddit:latest <user-login>/otus-reddit:1.0
~$ docker push <user-login>/otus-reddit:1.0 
The push refers to a repository [docker.io/dashishmakov/otus-reddit]
...
1.0: digest: sha256:6d60a3915efdd937a9ae0ad9e8c7b1fa2edc1a9d63000f9993cbcffa4fcca085 size: 3241


At the end remove the docker machine and romote instance
~$ docker-machine rm docker-host

