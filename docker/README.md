# Docker Notes

---

## Q. What is the init process? How does it work in Linux? Can we run Docker with init?

The init process is the **first process started by the Linux kernel** after it boots. It is the parent of all processes on the system.

- Always gets **PID 1**
- The kernel starts it directly — it is never spawned by another process
- Runs until the system shuts down
- If PID 1 dies → kernel panic, system crashes

```bash
ps -e | grep 1
# or
ps aux --pid 1
# output -> systemd
```

`systemd` does far more than just start processes — it manages the entire system lifecycle.

![systemd overview](https://github.com/user-attachments/assets/d7b293de-3ff6-4f0b-916d-a5ac41849d41)

---

### PID 1 inside a Docker container

```dockerfile
CMD ["node", "index.js"]   # node becomes PID 1 inside the container
```

**Problem:** `node` (or any app) is not designed to handle Linux signals or reap zombie processes as PID 1.

**Solution — use `tini` as a lightweight init inside containers:**

```dockerfile
FROM node:18
RUN apt-get install -y tini
ENTRYPOINT ["/usr/bin/tini", "--"]   # tini becomes PID 1, handles signals + zombies
CMD ["node", "index.js"]             # your app runs as PID 2
```

**Or use Docker's built-in init flag:**

```bash
docker run --init myapp    # Docker injects tini automatically as PID 1
```

---

## Q. What is build context? How is it used in Docker?

**Build context** is the directory Docker uses to locate files (source code, dependencies, etc.) for the build process. It can be the current path or a relative path.

```bash
# Current directory as build context
docker build -t imagedemo .

# Relative path as build context
docker build -t imagedemo /vss/build-path

# Custom Dockerfile path + custom build context path
docker build -t image-demo -f /path/to/custom-dockerfile /path/to/build-context
```

| Argument | Description |
|---|---|
| `.` | Build context is the current directory |
| `/vss/build-path` | Build context is a specific path |
| `-f /path/to/dockerfile` | Use a Dockerfile from a custom location |

---

## Q. Different Docker commands

| Command | Description |
|---|---|
| `docker ps -a` | List all containers (including stopped ones) |
| `docker container inspect <container_id>  or  docker inspect <container_id>` | Inspect container metadata |
| `docker container inspect <image_id>  or  docker inspect <image_id>` | Inspect image metadata |
| `docker top <container_id>` | List running processes inside a container |
| `docker container stop <image_id>  or  docker stop <container_id>` | Stop a specific container |
| `docker container start <image_id>  or  docker start <container_id>` | Start a stopped container |
| `docker stop $(docker ps -q)` | Stop all running containers |
| `docker container restart <image_id>  or  docker restart <container_id>` | Restart a container |
| `docker container rm <image_id>  or  docker rm <container_id>` | Delete a specific container |
| `docker container prune` | Delete all stopped containers |

| `docker volume create <volume-name> ` | Create a docker volume with specific name |
| `docker volume create ls ` | Show lists of volumes |
| `docker run -itd --name <container_name> --mount source=<volume-name-created-above>,target=/directory-name <image-name> ` | mount a specific volume on a particular directory in a container |
---

## Q. Write a Dockerfile to run a Flask application

**`vss_flask_app.py`** — simple Flask app to containerise.

**`Dockerfile`:**

```dockerfile
FROM python:3.10-slim

# Create a working directory inside the container
WORKDIR /vss_app

# COPY simply copies from source to destination (no extraction, no URL handling)
# ADD would also work here and can handle .tar extraction and URLs automatically:
#   ADD vss_flask_app.py /vss_app
COPY vss_flask_app.py /vss_app

# Install necessary libraries
# For Linux system packages: apt update && apt install -y nginx
RUN pip install flask && apt update && apt install -y build-essential

# Expose port 5000 inside the container
# Still need -p flag when running: docker run -p <hostPort>:<containerPort>
# e.g. docker run -p 5000:5000
EXPOSE 5000

CMD ["python", "vss_flask_app.py"]
```

> **`COPY` vs `ADD`**
> - `COPY` — simply copies files from source to destination. No extras.
> - `ADD` — does everything `COPY` does, plus auto-extracts `.tar` files and supports URLs.
> - Prefer `COPY` unless you specifically need extraction or URL support.

**Build and run:**

```bash
# Build the image
docker build -t vss-flask-app .

# Run the container with port mapping
docker run -p 5000:5000 vss-flask-app
```

![Dockerfile structure](https://github.com/user-attachments/assets/b8855aa8-cdf5-40c4-8ab5-6262c63bc9c1)

![Build output](https://github.com/user-attachments/assets/e673bda3-496f-4a70-b127-9dcdce615784)

![Running container](https://github.com/user-attachments/assets/23d0fb48-b207-4ece-a943-b273f79e1b67)

Q. can we delete docker images using imageId ?

Yes we can do but its not recommended to delete images using imageid insted we can delete using image tags one by one.

for many images the imageId can be same as the imageId doesn't change if SHA code is not changed during build.

docker rmi <tag of image>

docker rmi -f <imageId> -> deletes forcefully aal the images with same imageId.

Q. can we delete a image for a running container ?

 docker rmi dipti-img-v2:latest
Error response from daemon: conflict: unable to remove repository reference "dipti-img-v2:latest" (must force) - container 9dd9f58c362a is using its referenced image a64a85dd15b1

Docker protects you from deleting an image that a container (even a stopped one) depends on. The container must be removed first to release that reference.

force delete the image directly (not recommended)
bashdocker rmi -f dipti-img-v2:latest

⚠️ Force delete (-f) removes the image tag but the container still exists in a broken state — it loses its image reference. Always prefer stopping and removing the container cleanly first.

Q. How to create volume in docker and mount it to a container ? If the container is stopped how can we recover these files. can we attach that volume to another container?

docker volume create dipti-volume

docker run -itd --name dipti-container-1 --mount source=dipti-volume,target=/dipti-dir im-img:latest  ->   mount a volume on a particular directory in a container that is got created from  image im-img:latest created brfore.

enters into the containe and check the mount point created on directory /dipti-dir -

docker exec -it dipti-container-1 /bin/bash
df -f 
<img width="546" height="252" alt="image" src="https://github.com/user-attachments/assets/3092759a-1368-40ac-b9a4-c4c85d3500f8" />

you can create some files in that mount point and save. these files will also be stored in volume.
docker volume inspect dipti-volume -> it show the path of volumes. ( "Mountpoint": "/var/lib/docker/volumes/dipti-volume/_data"  )

Now if the container is stopped/teminated, we can get the date from above path in volume dipti-volume.

Attach this volume to another container and access those data-
docker run -itd --name dipti-cont-recov --mount source=dipti-volume,target=/test-recov-dir dipti-img-v2:latest
now enters into container dipti-cont-recov and check files in filesystem /test-recov-dir

Q. explain docker networking ? whats the default and custom networking ? what's the use case of custom networking.

When Docker is installed, it creates a virtual networking layer that allows containers to communicate with each other, the host, and the outside world. When you run any container without specifying a network, it joins the default bridge network (docker0).

Problem: Containers on the default bridge cannot resolve each other by name. You must use 'IP addresses', which change on every restart. so that you can't connect to other container with its name, instead you have to use their Ip which change on every restart which is diffecult.

but if we create custom networking, Containers now resolve each other by name automatically. Docker's built-in DNS resolver handles name resolution automatically on custom networks.

examples -

<img width="911" height="641" alt="image" src="https://github.com/user-attachments/assets/3997d135-fdbe-4d85-8090-c5a21092692c" />

The Default bridge Network — and Its Problem
When you run any container without specifying a network, it joins the default bridge network (docker0).
bashdocker run -d --name app1 nginx
docker run -d --name app2 nginx
Problem: Containers on the default bridge cannot resolve each other by name. You must use IP addresses, which change on every restart.
bash# This FAILS on default bridge
docker exec app1 ping app2   # ping: app2: Name or service not known
This is exactly why custom networks exist.

Creating and Using a Custom Network
Step 1 — Create a custom bridge network

docker network create my-custom-net -> it will create a custom network with subnet, ip-range, gateway. but if you want to define those specifically, then use below approach - 

docker network create \
  --driver bridge \
  --subnet 172.20.0.0/16 \
  --ip-range 172.20.240.0/20 \
  --gateway 172.20.0.1 \
  my-custom-net
Step 2 — Run containers on that network
bashdocker run -d --name backend --network my-custom-net my-backend-img
docker run -d --name frontend --network my-custom-net my-frontend-img

Step 3 — Containers now resolve each other by name automatically
bashdocker exec frontend ping backend       # Works!
docker exec frontend curl http://backend:5000/api   # Works!
Docker's built-in DNS resolver handles name resolution automatically on custom networks.
Useful network commands
bashdocker network ls                          # List all networks
docker network inspect my-custom-net       # Detailed info, connected containers
docker network connect my-custom-net app3  # Attach running container app3 to network my-custom-net
docker network disconnect my-custom-net app3
docker network rm my-custom-net            # Delete network

One of Use Case — Microservices Isolation
In a microservices setup, you don't want every service talking to every other service.

docker network create payments-net
docker network create inventory-net
docker network create notifications-net

# Payment service and its DB
docker run -d --name payment-svc --network payments-net payment-img
docker run -d --name payment-db  --network payments-net postgres

# Inventory service and its DB
docker run -d --name inventory-svc --network inventory-net inventory-img
docker run -d --name inventory-db  --network inventory-net postgres

# API Gateway connects to all services
docker network connect payments-net api-gateway
docker network connect inventory-net api-gateway
docker network connect notifications-net api-gateway

Q. Difference between CMD and ENTRYPOINT ?

CMD:-
=====

CMD is the default command that runs when the container starts.
It can be overriden using ' docker run <image_name> <commands>

e.g -> i created a dockerfile named as cmd and using ping command to ping to a website



