# CST8915 Lab 4: Introduction to Docker

| Student Name | Student ID | Course | Semester |
|---|---|---|---|
| Peng Wang | 041107730 | CST8915 Full-stack Cloud-native Development | Fall 2026 |

---

## Demo Video

🎥 [Watch Demo Video (Part 6: Docker Compose)](https://youtu.be/OfAECkPYsEc)

---

## Setup

| Item | Value |
|---|---|
| Docker host | Azure VM `docker-vm`, Standard_B2ls_v2 (2 vCPU, 4 GiB), Ubuntu 24.04 LTS, West US 2 |
| Docker | Docker Engine 29.9.0, Docker Compose v5.6.0, installed from Docker's apt repository |
| Access | VS Code Remote - SSH |

The lab asks for Standard_B2s. My Azure for Students subscription cannot create B-series v1 sizes, so I used Standard_B2ls_v2, which has the same 2 vCPUs and 4 GiB of memory.

---

## Reflection Questions

### 1. What are the main differences between a Docker image and a Docker container?

An image is a read-only template: a stack of layers with the base OS files, the dependencies, the application code and the start command. A container is a running instance created from an image, with its own writable layer, network interface and process. One image can start many containers. In Part 5, I started `app1`, `app2` and `app3` from the same `my-python-app:v3` image, and each one ran on its own port with its own files. An image never changes after it is built; to change it, I build a new image, as I did with v2 and v3. A container can be started, stopped and removed, and anything written inside it is lost when it is removed.

### 2. Explain how Docker's layered architecture improves efficiency.

Each Dockerfile instruction creates a read-only layer, and Docker reuses layers that have not changed. When I changed only `app.py` and built v3, the `WORKDIR`, `COPY requirements.txt` and `RUN pip install` steps were all `CACHED`. Only the `COPY app.py` layer was rebuilt, and the build took 0.8 seconds. Copying `requirements.txt` before the code makes this possible, because the slow dependency install is only repeated when the dependencies change. Layers are also shared on disk: `docker ps -s` showed each of my four Python containers using about 200 kB of its own, while the 160 MB of image layers was stored once and shared by all of them. Shared layers also mean less to download when pulling images.

### 3. Why does each container get its own writable layer?

The image layers are read-only and shared, so a container needs a private place to record its own changes. Docker adds a thin writable layer on top of the image for each container. In Part 5, `data1.txt` existed only in `app1`, `data2.txt` only in `app2`, and `data3.txt` only in `app3`. When I appended a line to `/app/app.py` in `app1`, copy-on-write copied the file into `app1`'s writable layer first, so `app2` still showed the original file. This keeps containers isolated from each other and keeps the image unchanged, so every new container starts from the same clean state. Because this layer is deleted with the container, data that must last, like the Redis counter in Part 6, belongs in a volume.

### 4. What are the benefits of using Docker Compose over running containers individually?

Compose describes the whole application in one `docker-compose.yml` file: the services, ports, environment variables, network and volume. One `docker compose up -d` built the web image, pulled Redis, created the network and the volume, and started both containers in the right order (`depends_on`). Without Compose, I would type several long `docker` commands and repeat them exactly every time. Compose also puts the services on one network where they find each other by name, which is why the Flask code can connect to `host='redis'`. Managing the app as a group is simple: `logs`, `stop`, `start`, `restart` and `down` act on all services at once, and `down` keeps the volume, so the counter continued from where it stopped. Scaling is one flag: `--scale web=3` started three web containers that shared the same Redis, and the counter kept counting across all three. The file is also easy to share and keep in Git.

---

## Challenges and Learnings (Optional)

- **Running `docker` inside a container.** After `docker exec -it my-nginx bash`, I typed `docker stop` and got `command not found`. The prompt had changed to `root@<container-id>`: I was inside the NGINX container, which does not have the Docker CLI. I typed `exit` first and ran the command on the VM.
- **Port 80 already in use.** The Part 3 container `python-app` still had port 80 when I reached Part 6, so I removed the old containers before `docker compose up`.
- **Slow `restart` and `down`.** Stopping the web service took about 10 seconds while Redis stopped at once. Docker sends SIGTERM, the Flask development server does not exit on it, and Docker waits 10 seconds before it kills the process.
- **Scaling needs dynamic host ports.** Three web containers cannot all bind host port 80, so for `--scale web=3` I changed the port mapping to `"5000"` and Docker gave each container a random host port (32771–32773).
- **`version` is obsolete.** Compose v5 warns that the `version: '3.8'` attribute is ignored. I kept the file as written in the lab; the warning does not affect the result.

---

## Acknowledgments

- Lab instructions: `ramymohamed10/26F_CST8915_Lab4`; Docker installation steps: [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/).
- GenAI declaration: As permitted for labs in this course, I used Claude (Anthropic) to explain the steps, check my portal settings and terminal output, draft the demo script, trim pauses and crop the borders of the demo video, and draft this README; I typed the commands, did the portal steps and recorded the demo myself.
