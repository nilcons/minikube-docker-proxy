# Minikube drivers

When using minikube, the user has to select a so called driver.

This selection governs how minikube starts the environment that is the worker/master node for the started minikube k8s cluster.

There are two possible choices: virtual machine or docker container.

Traditionally virtual machine was the only implemented solution, and the driver plugin system existed, because starting a virtual machine is different on every operating system.
On OSX one has to use qemu, on Windows hyperv, and on Linux kvm.
The problem with the virtual machine based drivers is that setup depends on admin privileges and usually collides with any kind of company security solution, like VPNs or firewalls.

Recently, minikube implemented the docker driver that runs a full worker+master node in a single container.
This only works if the system already has docker installed, but installation of Docker Desktop is usually allowed/simple even in locked down Windows/OSX environments.

This document focuses on making the docker driver experience as convenient as possible.

# Docker driver networking

Starting the minikube with the docker driver is simple, and requires no VM creation: `minikube start --driver=docker`

This creates a new docker network and a container in it (both called minikube):

```
$ docker ps
CONTAINER ID   IMAGE                      COMMAND                  CREATED          STATUS          PORTS                                                                                                                                  NAMES
ef9080e2d28b   kicbase/stable:v0.0.45     "/usr/local/bin/entr…"   11 hours ago     Up 28 minutes   127.0.0.1:32772->22/tcp, 127.0.0.1:32771->2376/tcp, 127.0.0.1:32770->5000/tcp, 127.0.0.1:32769->8443/tcp, 127.0.0.1:32768->32443/tcp   minikube
$ docker network inspect minikube
[
    {
        "Name": "minikube",
        "Driver": "bridge",
        "IPAM": {
            "Config": [
                {
                    "Subnet": "192.168.49.0/24",
                    "Gateway": "192.168.49.1"
                }
            ]
        },
        "Containers": {
            "ef9080e2d28bf8e483d0bd2e594801013999390cbf4bdb72590aaeccfceff84b": {
                "Name": "minikube",
                "EndpointID": "4969cc2f83c23bdd72321a4419aa2657bf06c9b8e9c42285aaea6293bc568c2a",
                "MacAddress": "02:42:c0:a8:31:02",
                "IPv4Address": "192.168.49.2/24",
                "IPv6Address": ""
            },
        },
    }
]
```

(The inspect output has been shortened, for readability.)

Everything one ever creates with `kubectl` will be neatly contained inside this new docker container called minikube.
All the internal kubernetes components (API server, controller manager, etc.) are also inside the minikube container.

The minikube network uses the bridge driver, and the Linux system that is running Docker is connected to this bridge, so the IP address of the minikube container can be used for accessing nodeport services and container hostports (e.g. the ingress):

```
$ minikube addons enable ingress
(wait for the installation)
$ kubectl get nodes -o wide
NAME       STATUS   ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
minikube   Ready    control-plane   10h   v1.31.0   192.168.49.2   <none>        Ubuntu 22.04.4 LTS   6.11.10-amd64    docker://27.2.0
$ kubectl describe deploy -n ingress-nginx ingress-nginx-controller
Name:                   ingress-nginx-controller
Namespace:              ingress-nginx
Pod Template:
  Containers:
   controller:
    Image:       registry.k8s.io/ingress-nginx/controller:v1.11.2@sha256:d5f8217feeac4887cb1ed21f27c2674e58be06bd8f5184cacea2a69abaf78dce
    Ports:       80/TCP, 443/TCP, 8443/TCP
    Host Ports:  80/TCP, 443/TCP, 0/TCP
```

(Describe output has been shortened for readability.)

Putting together the node IP and the ingress host port, one can visit http://192.168.49.2/ in a desktop browser and will see the default 404 from nginx.

# But this doesn't work on Windows/OSX

The last test only works if minikube is being ran on Linux, accessing the ingress doesn't work if Windows/OSX is used.

WHY?

Because the necessary kernel primitives for docker/kubernetes are only implemented on Linux, and therefore on Windows/OSX Docker Desktop creates a Linux VM for everything Docker related.

This VM (called Moby) is automatically  created and managed by Docker Desktop, and tries to stay out of the way as much as possible.
For example, Docker Desktop even implements port forwarding (with `docker run -p`), and volumes (with `docker run -v`).

One thing, Docker Desktop doesn't implement is routing from the OSX/Windows computer to the Linux VM.
Implementing this routing would collide with company private VPNs and firewalls, so they decided against it.

# Confirming the theory

Using a Windows/OSX one can confirm, that apart from the computer to minikube routing, everything still works correctly inside the Moby VM:

```
$ docker run -it --rm --network minikube nilcons/debian /bin/bash
root@50cdbf90b2d8:/# curl 192.168.49.2
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

By creating a new container inside the minikube network, one can send a request to the ingress with curl, and confirm that it works inside Moby.

Alas, working like this would restrict us to testing everything with curl only in text mode, and that's much less fun than seeing k8s deployed websites graphically in the browser.

# HTTP Proxy to the rescue

Let's create an HTTP proxy inside the minikube docker network:

```
$ docker run -it --rm --name proxy -p 8888:8888 --network minikube monokal/tinyproxy:latest ANY
[DockerTinyproxy][INFO][11:34:27]: DockerTinyproxy script started...
[DockerTinyproxy][INFO][11:34:27]: Checking for running Tinyproxy service...
[DockerTinyproxy][INFO][11:34:27]: Tinyproxy service not running.
[DockerTinyproxy][SUCCESS][11:34:27]: Allowed ANY - Edited /etc/tinyproxy/tinyproxy.conf successfully.
[DockerTinyproxy][INFO][11:34:27]: Starting Tinyproxy service...
[DockerTinyproxy][SUCCESS][11:34:27]: Tinyproxy service started successfully.
[DockerTinyproxy][INFO][11:34:27]: Tailing Tinyproxy log...
INFO      Dec 04 11:34:27 [20]: Initializing tinyproxy ...
INFO      Dec 04 11:34:27 [20]: Reloading config file
INFO      Dec 04 11:34:27 [20]: Setting "Via" header to 'tinyproxy'
...
```

Once this is running, minimize the terminal and keep it running while using minikube.

Now, change the browser's proxy settings to http://localhost:8888/, and minikube's 192.168.49.2 based URLs will start working, since the proxy is running at the same network environment as minikube.

Go to http://192.168.49.2/ to access the nginx ingress.

This trick will only work for HTTP based services, but core TCP/UDP based services can usually be easily tested in the command line without graphics, just use `docker run -it --rm --network minikube nilcons/debian` for those.

# Cleanup

Once finished working with minikube, stop the proxy with Ctrl-C (or with `docker rm -f proxy`).

Then the whole minikube cluster can be deleted with the usual `minikube delete`.

Don't forget to change back the browser's proxy settings to normal.
