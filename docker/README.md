# Docker Notes:-
--------------
Q. What is init process. how it works in linux...can we run docker with init?

The init process is the first process started by the Linux kernel after it boots. It is the parent of all processes on the system.

It always gets PID 1
The kernel starts it directly — it is never spawned by another process
It runs until the system shuts down
If PID 1 dies → kernel panic, system crashes

ps -e | grep 1 or ps aux --pid 1

o/p -> system

systemd does far more than just start processes — it manages the entire system lifecycle:
<img width="260" height="151" alt="image" src="https://github.com/user-attachments/assets/d7b293de-3ff6-4f0b-916d-a5ac41849d41" />

CMD ["node", "index.js"]   # node becomes PID 1 inside the container
Solution — use tini as a lightweight init inside containers:
FROM node:18

RUN apt-get install -y tini

ENTRYPOINT ["/usr/bin/tini", "--"]   # tini becomes PID 1, handles signals + zombies
CMD ["node", "index.js"]             # your app runs as PID 2

Or with Docker's built-in init:
bashdocker run --init myapp    # Docker injects tini automatically as PID 1

Q. what is build context? how its used in docker.
Build context is a directory where docker uses to locate files (source code, dependencies) for build process. it can be current path or relative path
docker build -t --name imagedemo . -> '.' is the build context in current directory
docker build -t --name imagedemo /vss/build-path ->  build context defined in relative path
docker build -t --name image-demo -f /path/to/custom-dockerfile /path/to/build-context -> build context defined in relative path and dockerfile reffered from relative path.

Q. Different docker commands -
List all containers (including stopped ones): docker ps -a
Inspect container metadata: docker inspect <container_id> or docker inspect <image_id>
List running processes in a container: docker top <container_id>
Stop a specific container: docker stop <container_id>
Start a stopped container: docker start <container_id>
Stop all running containers: docker stop $(docker ps -q)
Restart a container: docker restart <container_id>
Delete a specific container: docker rm <container_id>
Delete all stopped containers: docker container prune

Q. Write a docker file to run a flask application.

Docker file to run a python flask application in docker container

FROM python:3.10-slim

#create a working directory in container.
WORKDIR /vss_app

# it will copy the app.tar (compressed file) to container(/app) and will extract it. ADD automatically extracts and handles URL
#ADD vss_flask_app.py /vss_app -> it will work also
COPY vss_flask_app.py /vss_app

#Copies everything from current directory of host to container. (we can put 'COPY . /app'). Copy simply copies from source to destination without extraction and URL handling
#COPY . .

#install necessary libraries and configurations ( in Linux RUN apt intall && apt update -y nginx)
RUN pip install flask && apt update && apt install -y build-essential

#while running the container it exposes the application to port 5000. but while doing docker run we need to mention -p to open the port in host. ( -p <hostport>:<containerPort>  i.e -p 5000:80)
EXPOSE 5000

CMD [ "python","vss_flask_app.py"]

<img width="628" height="201" alt="image" src="https://github.com/user-attachments/assets/b8855aa8-cdf5-40c4-8ab5-6262c63bc9c1" />

<img width="1822" height="327" alt="image" src="https://github.com/user-attachments/assets/e673bda3-496f-4a70-b127-9dcdce615784" />
<img width="862" height="253" alt="image" src="https://github.com/user-attachments/assets/23d0fb48-b207-4ece-a943-b273f79e1b67" />

