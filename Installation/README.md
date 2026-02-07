<div align="center">
  <h1>Jenkins Installation</h1>
  <small>
    <strong>Author:</strong> Nguyễn Tấn Phát
  </small> <br />
  <sub>January 27, 2026</sub>
</div>

## Step 1. Directory Setup

Create a new folder on your computer (e.g., `jenkins-setup`) to hold the configuration files. Then you need to copy exactly **3 files** into that folder.

```text
jenkins-setup/
├── docker-compose.yaml
├── Dockerfile
└── plugins.txt
```

## Step 2. Execution Commands

Open your terminal (CMD or PowerShell) in the folder containing these files.

**1. Start and Build the System**

This command builds your custom image (installing all plugins) and starts the containers.

```bash
docker-compose up -d --build
```

- `-d`: Detached mode (runs in background).
- `--build`: Forces Docker to read your `Dockerfile` and `plugins.txt` again (use this whenever you update plugins).

**2. Access and Unlock Jenkins**

Browse to [http://localhost:8080](http://localhost:8080) (or whichever port you configured for Jenkins when installing it) and wait until the **Unlock Jenkins** page appears:

<p align="center">
  <img src="https://www.jenkins.io/doc/book/resources/tutorials/setup-jenkins-01-unlock-jenkins-page.jpg" style="width:70%;" alt="Unlock Jenkins">
</p>

Get the password in the console without having to open an interactive shell inside the container:

```bash
docker exec jenkins-blueocean cat /var/jenkins_home/secrets/initialAdminPassword
```

On the **Unlock Jenkins** page, paste this password into the **Administrator password** field and click **Continue**. Then create your first **Admin** user.

The setup wizard might ask to install plugins. Since we pre-installed them in the `Dockerfile`, you can select **"Install suggested plugins"** (it will be very fast as they are already cached) or skip it.

**3. Stop the System**

To stop the containers (data is preserved in volumes).

```bash
docker-compose down
```

**4. Reset/Wipe Data (Optional)**

**Note:** Use this only if you want to delete everything and start fresh.

```bash
docker-compose down -v
```
