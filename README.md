# Esercizio-2-Container


- Nella VMs creata nel punto 1 installare tramite Docker un'istanza Jenkins.
- Esporre Jenkins tramite un reverse proxy HaProxy (https://wiki.jenkins.io/display/JENKINS/Running+Jenkins+behind+HAProxy)
- Esporre tramite HaProxy l'apache web-server creato nel punto 1.

```
[root@localhost ~]# yum install docker-ce docker-ce-cli containerd.io
```

```
[root@localhost ~]# systemctl start docker
[root@localhost ~]# systemctl list-units | grep -i docker
sys-devices-virtual-net-docker0.device                                                   loaded active plugged   /sys/devices/virtual/net/docker0
sys-subsystem-net-devices-docker0.device                                                 loaded active plugged   /sys/subsystem/net/devices/docker0
docker.service                                                                           loaded active running   Docker Application Container Engine
docker.socket                                                                            loaded active running   Docker Socket for the API
[root@localhost ~]# systemctl enabled docker
Unknown operation 'enabled'.
[root@localhost ~]# systemctl enable docker
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
[root@localhost ~]# systemctl status docker
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2021-03-10 09:40:22 UTC; 32s ago
     Docs: https://docs.docker.com
 Main PID: 1459 (dockerd)
   CGroup: /system.slice/docker.service
           └─1459 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

Mar 10 09:40:21 localhost.localdomain dockerd[1459]: time="2021-03-10T09:40:21.830532250Z" level=info msg="scheme \"unix\" not registered, fallback to defau...dule=grpc
Mar 10 09:40:21 localhost.localdomain dockerd[1459]: time="2021-03-10T09:40:21.830567649Z" level=info msg="ccResolverWrapper: sending update to cc: {[{unix:...dule=grpc
Mar 10 09:40:21 localhost.localdomain dockerd[1459]: time="2021-03-10T09:40:21.830584605Z" level=info msg="ClientConn switching balancer to \"pick_first\"" module=grpc
Mar 10 09:40:21 localhost.localdomain dockerd[1459]: time="2021-03-10T09:40:21.862600189Z" level=info msg="Loading containers: start."
Mar 10 09:40:22 localhost.localdomain dockerd[1459]: time="2021-03-10T09:40:22.177412332Z" level=info msg="Default bridge (docker0) is assigned with an IP a... address"
Mar 10 09:40:22 localhost.localdomain dockerd[1459]: time="2021-03-10T09:40:22.245925004Z" level=info msg="Loading containers: done."
Mar 10 09:40:22 localhost.localdomain dockerd[1459]: time="2021-03-10T09:40:22.302504984Z" level=info msg="Docker daemon" commit=363e9a8 graphdriver(s)=over...n=20.10.5
Mar 10 09:40:22 localhost.localdomain dockerd[1459]: time="2021-03-10T09:40:22.302605133Z" level=info msg="Daemon has completed initialization"
Mar 10 09:40:22 localhost.localdomain systemd[1]: Started Docker Application Container Engine.
Mar 10 09:40:22 localhost.localdomain dockerd[1459]: time="2021-03-10T09:40:22.339595467Z" level=info msg="API listen on /var/run/docker.sock"
Hint: Some lines were ellipsized, use -l to show in full.
```

```
[root@localhost ~]# docker network create jenkins
```

```
[vagrant@localhost Jenkins]$ vi Dockerfile 
[vagrant@localhost Jenkins]$ ls
Dockerfile

[vagrant@localhost Jenkins]$ cat Dockerfile 
FROM jenkins/jenkins:2.263.4-lts-jdk11
USER root
RUN apt-get update && apt-get install -y apt-transport-https \
       ca-certificates curl gnupg2 \
       software-properties-common
RUN curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add -
RUN apt-key fingerprint 0EBFCD88
RUN add-apt-repository \
       "deb [arch=amd64] https://download.docker.com/linux/debian \
       $(lsb_release -cs) stable"
RUN apt-get update && apt-get install -y docker-ce-cli
USER jenkins
RUN jenkins-plugin-cli --plugins blueocean:1.24.4
```

```
[vagrant@localhost Jenkins]$ sudo docker build -t myjenkins-blueocean:1.1 .
```

```
[vagrant@localhost Jenkins]$ docker run \
>   --name jenkins-blueocean \
>   --rm \
>   --detach \
>   --network jenkins \
>   --env DOCKER_HOST=tcp://docker:2376 \
>   --env DOCKER_CERT_PATH=/certs/client \
>   --env DOCKER_TLS_VERIFY=1 \
>   --publish 8080:8080 \
>   --publish 50000:50000 \
>   --volume jenkins-data:/var/jenkins_home \
>   --volume jenkins-docker-certs:/certs/client:ro \
>   myjenkins-blueocean:1.1 

```

```
[vagrant@localhost Jenkins]$ sudo docker ps -a
CONTAINER ID   IMAGE                     COMMAND                  CREATED          STATUS          PORTS                                              NAMES
2c1cc007c268   myjenkins-blueocean:1.1   "/sbin/tini -- /usr/…"   35 seconds ago   Up 33 seconds   0.0.0.0:8080->8080/tcp, 0.0.0.0:50000->50000/tcp   jenkins-blueocean
```

seguendo la documentazione ho fatto:

```
[vagrant@localhost Jenkins]$ sudo docker logs 2c1cc007c268

*************************************************************
*************************************************************
*************************************************************

Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

60402fa7d4934fc68f9bbeccf75a3008

This may also be found at: /var/jenkins_home/secrets/initialAdminPassword

*************************************************************
*************************************************************
*************************************************************
```

http://192.168.1.215:8080/login?from=%2F


dopo aver installato i pacchetti essenziali mi ritrovo alla pagina principale di jenkins!

Welcome to Jenkins!
This page is where your Jenkins jobs will be displayed. To get started, you can set up distributed builds or start building a software project.


