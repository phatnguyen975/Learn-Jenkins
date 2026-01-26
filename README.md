<div align="center">
  <h1>Jenkins Tutorial</h1>
  <small>
    <strong>Author:</strong> Nguyen Tan Phat
  </small> <br />
  <sub>January 27, 2026</sub>
  <div>
    Follow the full video tutorial on
    <a href="https://www.youtube.com/watch?v=6YZvp2GwT0A" target="_blank"><b>YouTube</b></a>
  </div>
</div>

## Table of Contents

1. [Jenkins Overview](#1-jenkins-overview)
2. [Installation with Docker](#2-installation-with-docker)

## 1. Jenkins Overview

### What is Jenkins?

Jenkins is an open-source **Automation Server** (written in Java). It is the heart of the CI/CD process.

- **Simplification:** Automates repetitive tasks as building code, running tests, deploying to servers.
- **Why use Jenkins?**
  - **Early Bug Detection:** Automatically runs tests every time code is committed.
  - **Time Saving:** No more manual builds.
  - **Huge Ecosystem**: Thousands of plugins (Git, Docker, Kubernetes, Slack, Email, etc.).

### Core Concepts

Before installation, you must understand these keywords:

- **Job / Project:** A specific CI/CD task (e.g., "Build Project A").
- **Pipeline:** A workflow of sequential steps (Code -> Build -> Test -> Deploy). This is the modern standard.
- **Node / Agent:** The machine (or container) where Jenkins executes the work.
  - **Controller (Master):** Manages the schedule and distributes tasks.
  - **Agent (Slave):** The "worker" that performs the heavy lifting (building, testing).
- **Executor:** The number of concurrent threads (slots) for processing jobs on a Node.
- **Workspace:** A temporary directory on the disk where Jenkins checks out source code and builds files.

## 2. Installation with Docker

### Why Run Jenkins with Docker?

- No manual Java setup
- Fast installation
- Easy backup & migration
- Environment consistency
- Docker = Jenkins runs the same everywhere

### Installation

1. Create a Dockerfile with the following content:

- **Windows:**

```dockerfile
FROM jenkins/jenkins:2.541.1-jdk21
USER root
RUN apt-get update && apt-get install -y lsb-release
RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
    https://download.docker.com/linux/debian/gpg
RUN echo "deb [arch=$(dpkg --print-architecture) \
    signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
    https://download.docker.com/linux/debian \
    $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
RUN apt-get update && apt-get install -y docker-ce-cli
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean docker-workflow json-path-api"
```

- **MacOS and Linux:**

```dockerfile
FROM jenkins/jenkins:2.541.1-jdk21
USER root
RUN apt-get update && apt-get install -y lsb-release ca-certificates curl && \
    install -m 0755 -d /etc/apt/keyrings && \
    curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc && \
    chmod a+r /etc/apt/keyrings/docker.asc && \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
    https://download.docker.com/linux/debian $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" \
    | tee /etc/apt/sources.list.d/docker.list > /dev/null && \
    apt-get update && apt-get install -y docker-ce-cli && \
    apt-get clean && rm -rf /var/lib/apt/lists/*
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean docker-workflow json-path-api"
```

2. Build a new docker image from this Dockerfile and assign the image a meaningful name, e.g. `myjenkins-blueocean:2.541.1`:

```bash
docker build -t myjenkins-blueocean:2.541.1 .
```

3. Create a bridge network in Docker:

```bash
docker network create jenkins
```

4. Run your own `myjenkins-blueocean:2.541.1` image as a container in Docker:

- **Windows:**

```bash
docker run --name jenkins-blueocean --restart=on-failure --detach ^
  --network jenkins --env DOCKER_HOST=tcp://docker:2376 ^
  --env DOCKER_CERT_PATH=/certs/client --env DOCKER_TLS_VERIFY=1 ^
  --volume jenkins-data:/var/jenkins_home ^
  --volume jenkins-docker-certs:/certs/client:ro ^
  --publish 8080:8080 --publish 50000:50000 myjenkins-blueocean:2.541.1
```

- **MacOS and Linux:**

```bash
docker run --name jenkins-blueocean --restart=on-failure --detach \
  --network jenkins --env DOCKER_HOST=tcp://docker:2376 \
  --env DOCKER_CERT_PATH=/certs/client --env DOCKER_TLS_VERIFY=1 \
  --publish 8080:8080 --publish 50000:50000 \
  --volume jenkins-data:/var/jenkins_home \
  --volume jenkins-docker-certs:/certs/client:ro \
  myjenkins-blueocean:2.541.1
```

5. Browse to `http://localhost:8080` (or whichever port you configured for Jenkins when installing it) and wait until the **Unlock Jenkins** page appears:

<p align="center">
  <img src="https://www.jenkins.io/doc/book/resources/tutorials/setup-jenkins-01-unlock-jenkins-page.jpg" style="width:60%;" alt="Unlock Jenkins">
</p>

6. Get the password in the console without having to open an interactive shell inside the container:

```bash
docker exec jenkins-blueocean cat /var/jenkins_home/secrets/initialAdminPassword
```

7. On the **Unlock Jenkins** page, paste this password into the **Administrator password** field and click **Continue**. Then create your first **Admin** account.
