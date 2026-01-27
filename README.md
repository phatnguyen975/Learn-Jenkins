<div align="center">
  <h1>Jenkins Tutorial</h1>
  <small>
    <strong>Author:</strong> Nguyen Tan Phat
  </small> <br />
  <sub>January 27, 2026</sub>
  <div>
    Follow the full video tutorial on
    <a href="https://www.youtube.com/watch?v=OXP8YBPBdgw" target="_blank"><b>YouTube</b></a>
  </div>
</div>

## Table of Contents

1. [Jenkins Overview](#1-jenkins-overview)
2. [Installation with Docker](#2-installation-with-docker)
3. [Jenkins Pipeline](#3-jenkins-pipeline)
4. [Pipeline Triggers](#4-pipeline-triggers)

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

1. Create a bridge network in Docker:

```bash
docker network create jenkins
```

2. Run a `docker:dind` Docker image:

- **Windows:**

```bash
docker run --name jenkins-docker --rm --detach `
  --privileged --network jenkins --network-alias docker `
  --env DOCKER_TLS_CERTDIR=/certs `
  --volume jenkins-docker-certs:/certs/client `
  --volume jenkins-data:/var/jenkins_home `
  --publish 2376:2376 `
  docker:dind
```

- **MacOS and Linux:**

```bash
docker run --name jenkins-docker --rm --detach \
  --privileged --network jenkins --network-alias docker \
  --env DOCKER_TLS_CERTDIR=/certs \
  --volume jenkins-docker-certs:/certs/client \
  --volume jenkins-data:/var/jenkins_home \
  --publish 2376:2376 \
  docker:dind --storage-driver overlay2
```

3. Create a Dockerfile with the following content:

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

4. Build a new docker image from this Dockerfile and assign the image a meaningful name, e.g. `myjenkins-blueocean:2.541.1`:

```bash
docker build -t myjenkins-blueocean:2.541.1 .
```

5. Run your own `myjenkins-blueocean:2.541.1` image as a container in Docker:

- **Windows:**

```bash
docker run --name jenkins-blueocean --restart=on-failure --detach `
  --network jenkins --env DOCKER_HOST=tcp://docker:2376 `
  --env DOCKER_CERT_PATH=/certs/client --env DOCKER_TLS_VERIFY=1 `
  --volume jenkins-data:/var/jenkins_home `
  --volume jenkins-docker-certs:/certs/client:ro `
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

6. Browse to [http://localhost:8080](http://localhost:8080) (or whichever port you configured for Jenkins when installing it) and wait until the **Unlock Jenkins** page appears:

<p align="center">
  <img src="https://www.jenkins.io/doc/book/resources/tutorials/setup-jenkins-01-unlock-jenkins-page.jpg" style="width:70%;" alt="Unlock Jenkins">
</p>

7. Get the password in the console without having to open an interactive shell inside the container:

```bash
docker exec jenkins-blueocean cat /var/jenkins_home/secrets/initialAdminPassword
```

8. On the **Unlock Jenkins** page, paste this password into the **Administrator password** field and click **Continue**. Then create your first **Admin** user.

## 3. Jenkins Pipeline

Traditionally, Jenkins jobs were created by clicking through the UI (Freestyle projects). This was hard to version control and hard to backup. **Pipeline as Code** means you write a text file called `Jenkinsfile` and commit it to your Git repository. The entire build process is versioned just like your application code.

There are two syntaxes in Jenkins:

- **Declarative Pipeline:** (Recommended) Newer, stricter structure, easier to read, better error handling. We will focus on this.
- **Scripted Pipeline:** Older, based strictly on Groovy. Very powerful but harder to maintain.

Basic declarative pipeline structure:

```groovy
pipeline {
    agent any // Run on any available agent

    stages {
        stage('Checkout') {
            steps {
                git branch 'main', url: 'https://github.com/example/repo.git' // Source code checkout
            }
        }
        stage('Build') {
            steps {
                echo 'Building...'
                sh './mvnw clean package' // Shell command to build
            }
        }
        stage('Test') {
            steps {
                echo 'Testing...'
                sh './mvnw test'
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploying...'
            }
        }
    }
}
```

- `pipeline { ... }`: Root block wrapping the entire process.
- `agent any`: Defines where the job runs. `any` means Jenkins will find an available executor.
- `stages { ... }`: Contains the list of stages.
- `stage('Stage Name')`: Defines a specific phase (Build, Test, Deploy, etc.).
- `steps { ... }`: The actual tasks performed in that stage.
- `sh '...'`: Executes a Shell command (Linux). Use `bat '...'` for Windows agents.

### Anatomy of a Declarative Pipeline

1. **The `agent` Directive**

This defines **where** the pipeline (or a specific stage) runs.

- `agent any`: Runs on the first available executor on any node.
- `agent none`: Prevents a global agent from being allocated. Useful if you want to define agents individually for each stage.
- `agent { label 'linux-machine' }`: Runs on a specific node labeled 'linux-machine'.
- `agent { docker ... }`: (Most useful) Spins up a Docker container, runs the steps inside it, and destroys it afterwards.

```groovy
pipeline {
    agent {
        docker {
            image 'maven:3.8.1-adoptopenjdk-11'
            args '-v /tmp/m2:/root/.m2' // Cache maven dependencies
        }
    }

    stages { ... }
}
```

2. **The `environment` Directive**

Defines environment variables. You can define them globally (top of pipeline) or per stage.

```groovy
pipeline {
    agent any

    environment {
        // Global variable available to all stages
        CC = 'clang'
        DB_PASSWORD = credentials('my-db-password') // Secure injection
    }

    stages {
        stage('Deploy') {
            environment {
                // Local variable, only available in this stage
                staging_url = "staging.example.com"
            }
            steps {
                sh 'echo Deploying to $staging_url using DB pass $DB_PASSWORD'
            }
        }
    }
}
```

3. **The `parameters` Directive**

Allows you to parameterize your build. When you click **Build**, Jenkins will ask for these inputs.

```groovy
pipeline {
    agent any

    parameters {
        string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who to greet?')
        booleanParam(name: 'DEPLOY_TO_PROD', defaultValue: false, description: 'Deploy to Production?')
        choice(name: 'ENV', choices: ['dev', 'test', 'prod'], description: 'Select Environment')
    }

    stages {
        stage('Greeting') {
            steps {
                echo "Hello ${params.PERSON}"
                echo "Target Environment: ${params.ENV}"
            }
        }
    }
}
```

### Control Flow and Logic

1. **The `when` Directive (Conditionals)**

Skips a stage unless a condition is met.

- **Usage:** Inside a `stage` block.
- **Common Conditions:** `branch`, `tag`, `environment`, `expression` (groovy code), `changeRequest` (Pull Request).

```groovy
stage('Deploy to Production') {
    when {
        allOf {
            branch 'main' // Only run on 'main' branch
            expression { params.DEPLOY_TO_PROD == true } // And if user checked the box
        }
    }
    steps {
        echo 'Deploying to Production...'
    }
}
```

2. **The `parallel` Directive (Speed)**

Runs multiple sub-stages at the same time. Great for running different types of tests simultaneously to save time.

```groovy
stage('Quality Checks') {
    parallel {
        stage('Unit Tests') {
            steps { sh './mvnw test' }
        }
        stage('Integration Tests') {
            steps { sh './run-integration-tests.sh' }
        }
        stage('Security Scan') {
            steps { sh './security-scanner' }
        }
    }
}
```

3. **The `input` Directive (Manual Approval)**

Pauses the pipeline and waits for a human to click **Proceed** or **Abort**. Essential for Continuous Delivery (CD) to production.

```groovy
stage('Deploy to Prod') {
    input {
        message "Should we deploy to production?"
        ok "Yes, do it!"
        submitter "admin-user" // Only allow specific user to approve
    }
    steps {
        sh './deploy.sh'
    }
}
```

### Handling Failures and Notifications

**Default behavior:** Pipeline stops when a step fails.

1. **Custom Handling**

```groovy
stage('Test') {
    steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
            sh 'npm test'
        }
    }
}
```

2. **The `post` Directive**

This block runs at the end of the Pipeline (or stage), regardless of what happened in the `steps`.

- `always`: Runs regardless of status.
- `success`: Only if build succeeded.
- `failure`: Only if build failed.
- `unstable`: Only if test results were bad, but the build finished.
- `changed`: Only if the status changed from the previous run (e.g., fixed -> broken).

```groovy
post {
    always {
        junit 'target/surefire-reports/*.xml' // Record test results
        cleanWs() // Clean workspace to save disk space
    }
    failure {
        mail to: 'team@example.com',
             subject: "Build Failed: ${currentBuild.fullDisplayName}",
             body: "Something went wrong. Check logs."
    }
}
```

### Other Useful Directives

1. **The `options` Directive**

Configures job-specific settings within the Pipeline itself.

```groovy
options {
    timeout(time: 1, unit: 'HOURS') // Abort if running longer than 1 hour
    retry(3) // Retry the whole pipeline 3 times if it fails
    timestamps() // Add timestamps to console log output
    disableConcurrentBuilds() // Prevent multiple builds from running at once
}
```

2. **The `tools` Directive**

Automatically installs tools (defined in "Global Tool Configuration") on the agent.

```groovy
tools {
    maven 'Maven 3.8.1'
    jdk 'Java 17'
}
```

3. **The `script` Directive**

It acts as an "escape hatch," allowing you to execute arbitrary **Scripted Pipeline code (Groovy)** inside the strict structure of a Declarative Pipeline. The script block bridges this gap by providing the full power of the Groovy language when needed.

The `script` block is always defined inside a `steps` block.

You should use the `script` directive in the following specific scenarios where standard Declarative syntax is insufficient.

- **Assigning Command Output to Variables:** In standard Declarative syntax, you cannot easily capture the output of a shell command into a variable. The `script` block solves this.

```groovy
stage('Capture Output') {
    steps {
        script {
            // 'returnStdout: true' returns the output instead of printing it
            // '.trim()' removes trailing newlines
            def commitHash = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()

            // Now you can use this variable globally in the script block
            echo "Current Commit: ${commitHash}"
            env.MY_IMAGE_TAG = commitHash // Export to environment variable
        }
    }
}
```

- **Loops and Iterations (`for`, `while`):** Declarative Pipelines do not support loops natively. To repeat a task (e.g., deploying to multiple servers), you must use a `script` block.

```groovy
stage('Deploy to Multiple Servers') {
    steps {
        script {
            def servers = ['192.168.1.10', '192.168.1.20', '192.168.1.30']

            for (server in servers) {
                echo "Deploying to ${server}..."
                // sh "ssh user@${server} ..."
            }
        }
    }
}
```

- **Advanced Exception Handling (`try-catch`):** While Declarative Pipelines have a `post` section for general status handling, they cannot handle non-fatal errors within a specific step. Use `script` for granular `try-catch` logic.

```groovy
stage('Optional Cleanup') {
    steps {
        script {
            try {
                sh './risky-cleanup-script.sh'
            } catch (Exception e) {
                echo "Cleanup failed, but proceeding with the build..."
                echo "Error message: ${e.getMessage()}"
                // The build will remain 'SUCCESS' instead of failing
            }
        }
    }
}
```

- **Complex Conditional Logic (`if-else`):** Declarative Pipelines provide the `when` directive for skipping entire stages. However, if you need conditional logic inside a **single** step, `script` is required.

```groovy
stage('Logic Check') {
    steps {
        script {
            if (env.BRANCH_NAME == 'main' || env.BRANCH_NAME == 'production') {
                echo "Running comprehensive test suite..."
                sh './run-full-tests.sh'
            } else {
                echo "Running smoke tests only..."
                sh './run-smoke-tests.sh'
            }
        }
    }
}
```

## 4. Pipeline Triggers

Triggers define when a pipeline should start automatically. Without triggers, you have to click **Build Now** manually. You define these in the `triggers { ... }` directive.

### `cron` (Time-based Trigger)

Runs the job periodically at a specific time (like Linux Crontab).

- **Syntax:** `MINUTE HOUR DOM MONTH DOW`
  - `MINUTE` - Minutes within the hour (0-59)
  - `HOUR` - The hour of the day (0-23)
  - `DOM` - The day of the month (1-31)
  - `MONTH` - The month (1-12)
  - `DOW` - The day of the week (0-7) where 0 and 7 are Sunday
- **Note:** Use `H` (Hash) instead of exact numbers to distribute load. Jenkins will pick a random minute (e.g., `H` might become 12, 34, or 55) to prevent all jobs from starting at exactly 00:00.

```groovy
pipeline {
    agent any
    triggers {
        // Run every morning at a random minute between 4:00 AM and 4:59 AM
        cron('H 4 * * *')
    }

    stages { ... }
}
```

### `pollSCM` (Polling Git)

Jenkins checks the Git repository periodically to see if there are new commits. If code changes are detected, it builds.

- **Pros:** Easy to set up, no network configuration needed on GitHub side.
- **Cons:** Inefficient (waste of resources checking constantly). Slow (delay between push and build).

```groovy
pipeline {
    agent any
    triggers {
        // Check Git for changes every 15 minutes
        pollSCM('H/15 * * * *')
    }

    stages { ... }
}
```

### `upstream` (Job Chaining)

Starts this pipeline automatically after another job finishes successfully. This is useful for chaining microservices (e.g., "Build Backend" -> finishes -> triggers "Integration Test").

```groovy
pipeline {
    agent any
    triggers {
        // Run this job if 'backend-api-build' finishes successfully
        upstream(upstreamProjects: 'backend-api-build', threshold: hudson.model.Result.SUCCESS)
    }

    stages { ... }
}
```
