<div align="center">
  <h1>Jenkins Tutorial</h1>
  <small>
    <strong>Author:</strong> Nguyễn Tấn Phát
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
5. [References](#5-references)

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

### Why Use Pipeline?

While standard Jenkins "freestyle" jobs support simple CI by allowing you to define sequential tasks in an application lifecycle, they do not create a persistent record of execution, enable one script to address all the steps in a complex workflow, or confer the other advantages of pipelines.

In contrast to freestyle jobs, pipelines enable you to define the whole application lifecycle. Pipeline functionality helps Jenkins to support continuous delivery (CD). The Pipeline plugin was built with requirements for a flexible, extensible, and script-based CD workflow capability in mind.

Accordingly, pipeline functionality is:

- **Durable:** Pipelines can survive both planned and unplanned restarts of your Jenkins controller.
- **Pausable:** Pipelines can optionally stop and wait for human input or approval before completing the jobs for which they were built.
- **Versatile:** Pipelines support complex real-world CD requirements, including the ability to fork or join, loop, and work in parallel with each other.
- **Efficient:** Pipelines can restart from any of several saved checkpoints.
- **Extensible:** The Pipeline plugin supports custom extensions to its DSL (domain scripting language) and multiple options for integration with other plugins.

The flowchart below is an example of one continuous delivery scenario enabled by the Pipeline plugin:

<p align="center">
  <img src="https://www.jenkins.io/images/pipeline/jenkins-workflow.png" style="width:70%;" alt="Jenkins Pipeline">
</p>

## 2. Installation with Docker

### Why Run Jenkins with Docker?

- No manual Java setup
- Fast installation
- Easy backup & migration
- Environment consistency
- Docker = Jenkins runs the same everywhere

### Installation with Docker Compose

You can see the step-by-step instructions at [Installation with Docker Compose](./Installation/README.md).

### Installation with Jenkins Documentation

You can see the step-by-step instructions at [Installation with Jenkins Documentation](https://www.jenkins.io/doc/book/installing/docker/).

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

## 5. References

- [Jenkins Pipeline Tutorial](https://www.jenkins.io/pipeline/getting-started-pipelines/)
- [Jenkins Basics To Production](https://github.com/CloudWithVarJosh/Jenkins-Basics-To-Production/tree/main)
- [Jenkins Plugins](https://plugins.jenkins.io/ui/search/)
