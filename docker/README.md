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
| `docker inspect <container_id>` | Inspect container metadata |
| `docker inspect <image_id>` | Inspect image metadata |
| `docker top <container_id>` | List running processes inside a container |
| `docker stop <container_id>` | Stop a specific container |
| `docker start <container_id>` | Start a stopped container |
| `docker stop $(docker ps -q)` | Stop all running containers |
| `docker restart <container_id>` | Restart a container |
| `docker rm <container_id>` | Delete a specific container |
| `docker container prune` | Delete all stopped containers |

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
