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

## ho creato un altro box e installato haproxy

sudo yum install haproxy

cd /etc/haproxy

vi haproxy.cfg

```
[root@localhost haproxy]# cat haproxy.cfg 
# If you already have an haproxy.cfg file, you can probably leave the
# global and defaults section as-is, but you might need to increase the
# timeouts so that long-running CLI commands will work.
global
    maxconn 4096
    log 127.0.0.1 local0 debug

defaults
   log global
   option httplog
   option dontlognull
   option forwardfor
   maxconn 20
   timeout connect 5s
   timeout client 60s
   timeout server 60s

frontend http-in
   bind *:80
   mode http
   acl prefixed-with-jenkins  path_beg /jenkins/
   acl host-is-jenkins-example   hdr(host) eq jenkins.example.com
   use_backend jenkins if host-is-jenkins-example prefixed-with-jenkins

backend jenkins
   server jenkins1 192.168.1.215:8080
   mode http
   reqrep ^([^\ :]*)\ /(.*) \1\ /\2
   acl response-is-redirect res.hdr(Location) -m found
   # Must combine following two lines into a SINGLE LINE for HAProxy
   rspirep ^Location:\ (http|https)://192.168.1.215:8080/jenkins/(.*) Location:\ \1://jenkins.example.com/jenkins/\2 if response-is-redirect
```

## ho editato il file seguendo la configurazione, mettendo l'indirizzo ip 192.168.1.215 dell'altra macchina vagrant dove gira jenkins con docker

```
[vagrant@localhost haproxy]$ sudo systemctl enable haproxy
Created symlink from /etc/systemd/system/multi-user.target.wants/haproxy.service to /usr/lib/systemd/system/haproxy.service.
```

```
[vagrant@localhost haproxy]$ sudo systemctl status haproxy
● haproxy.service - HAProxy Load Balancer
   Loaded: loaded (/usr/lib/systemd/system/haproxy.service; disabled; vendor preset: disabled)
   Active: inactive (dead)
[vagrant@localhost haproxy]$ sudo systemctl start haproxy
[vagrant@localhost haproxy]$ sudo systemctl status haproxy
● haproxy.service - HAProxy Load Balancer
   Loaded: loaded (/usr/lib/systemd/system/haproxy.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2021-03-11 14:39:37 UTC; 3s ago
 Main PID: 1139 (haproxy-systemd)
   CGroup: /system.slice/haproxy.service
           ├─1139 /usr/sbin/haproxy-systemd-wrapper -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid
           ├─1141 /usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -Ds
           └─1142 /usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -Ds

Mar 11 14:39:37 localhost.localdomain systemd[1]: Stopped HAProxy Load Balancer.
Mar 11 14:39:37 localhost.localdomain systemd[1]: Started HAProxy Load Balancer.
Mar 11 14:39:37 localhost.localdomain haproxy-systemd-wrapper[1139]: haproxy-systemd-wrapper: executing /usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -p /run/...pid -Ds
Hint: Some lines were ellipsized, use -l to show in full.
```


## haproxy sta ascoltando su tutte le interfacce...

```
[vagrant@localhost haproxy]$ sudo netstat -antp
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      1142/haproxy        
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      673/sshd            
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      893/master          
tcp        0      0 0.0.0.0:111             0.0.0.0:*               LISTEN      370/rpcbind         
tcp        0      0 10.0.2.15:22            10.0.2.2:35130          ESTABLISHED 911/sshd: vagrant [ 
tcp6       0      0 :::22                   :::*                    LISTEN      673/sshd            
tcp6       0      0 ::1:25                  :::*                    LISTEN      893/master          
tcp6       0      0 :::111                  :::*                    LISTEN      370/rpcbind    


[vagrant@localhost haproxy]$ ifconfig
eth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.199  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::18e7:337c:e06f:a050  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:02:20:22  txqueuelen 1000  (Ethernet)
        RX packets 294  bytes 20908 (20.4 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 117  bytes 10608 (10.3 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```


però per qualche motivo haproxy non redirige il traffico verso l'host di jenkins installato sull'altra macchina vagrant
