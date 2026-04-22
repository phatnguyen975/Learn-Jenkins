def CHANGED_SERVICES = ""
def IS_ROOT_CHANGED = false
def BUILD_BACKOFFICE = false
def BUILD_STOREFRONT = false

def VALID_BACKEND_SERVICES = [
    'media', 'product', 'cart', 'order', 'rating', 'customer',
    'location', 'inventory', 'tax', 'search', 'recommendation',
    'payment', 'payment-paypal', 'sampledata', 'webhook',
    'promotion', 'backoffice-bff', 'storefront-bff'
]

def VALID_FRONTEND_APP_NAME = [
    'backoffice-ui': 'backoffice',
    'storefront-ui': 'storefront'
]

def VALID_FRONTEND_IMAGE_NAME = [
    'backoffice-ui': 'yas-backoffice',
    'storefront-ui': 'yas-storefront'
]

def resolveBackendServices(boolean isRootChanged, String changedServices) {
    return isRootChanged ? VALID_BACKEND_SERVICES : changedServices.split(',').findAll { it?.trim() }
}

def resolveFrontendServices(boolean isRootChanged, boolean buildBackoffice, boolean buildStorefront) {
    if (isRootChanged) {
        return VALID_FRONTEND_APP_NAME.keySet() as List
    }

    def frontendServices = []
    if (buildBackoffice) {
        frontendServices << 'backoffice-ui'
    }
    if (buildStorefront) {
        frontendServices << 'storefront-ui'
    }

    return frontendServices
}

def processCoverage(String service) {
    recordCoverage(
        id: "coverage-${service}",
        name: "Coverage: ${service.capitalize()}",
        tools: [[
            parser: 'JACOCO',
            pattern: "${service}/target/site/jacoco/jacoco.xml"
        ]],
        qualityGates: [
            [threshold: 70.0, metric: 'LINE', baseline: 'PROJECT', criticality: 'UNSTABLE'],
            [threshold: 70.0, metric: 'BRANCH', baseline: 'PROJECT', criticality: 'FAILURE'],
            [threshold: 70.0, metric: 'INSTRUCTION', baseline: 'PROJECT', criticality: 'FAILURE'],
            [threshold: 70.0, metric: 'METHOD', baseline: 'PROJECT', criticality: 'UNSTABLE'],
            [threshold: 70.0, metric: 'CLASS', baseline: 'PROJECT', criticality: 'FAILURE']
        ]
    )
}

def runBackendSonarQube(String service) {
    withSonarQubeEnv('SonarQube-Local') {
        echo ">>> SonarQube scanning: ${service}"
        dir(service) {
            sh "mvn sonar:sonar -Dmaven.repo.local=${WORKSPACE}/.m2/repository"
        }
    }
}

def runBackendSnyk(String service) {
    withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
        echo ">>> Snyk scanning: ${service}"
        dir(service) {
            sh 'chmod +x ./mvnw'
            if (env.BRANCH_NAME == 'main') {
                sh "npx snyk monitor --project-name=yas-${service}"
            }
            sh "npx snyk test --severity-threshold=high"
        }
    }
}

def cleanupLocalM2Repo(int maxSizeGb = 3) {
    sh """
        if [ -d .m2/repository ]; then
            size_mb=\$(du -sm .m2/repository | cut -f1)
            limit_mb=\$(( ${maxSizeGb} * 1024 ))
            echo "Local .m2 cache size: \${size_mb} MB (limit: \${limit_mb} MB)"
            if [ "\${size_mb}" -gt "\${limit_mb}" ]; then
                echo "Local .m2 cache exceeds limit; cleaning .m2/repository"
                rm -rf .m2/repository
            fi
        fi
    """
}

def runFrontendPipeline(String service) {
    def appName = VALID_FRONTEND_APP_NAME[service]
    if (!appName) {
        error("Unsupported frontend service: ${service}")
    }

    dir(appName) {
        echo "Installing ${appName} dependencies..."
        sh 'npm ci'

        echo "Checking code quality (Linting)..."
        sh 'npm run lint'

        echo "Running SonarQube analysis for ${appName}..."
        withSonarQubeEnv('SonarQube-Local') {
            def scannerHome = tool 'SonarScanner'
            sh "${scannerHome}/bin/sonar-scanner"
        }

        echo "Scanning ${appName} dependencies..."
        withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
            if (env.BRANCH_NAME == 'main') {
                sh "npx snyk monitor --project-name=yas-${appName}"
            }
            sh "npx snyk test --severity-threshold=high"
        }

        echo "Building ${appName} UI..."
        sh 'npm run build'
    }
}

def runBackendPipelineAndPush(String service, String dockerNamespace, String dockerCredentialsId) {
    echo "Building and testing ${service}..."
    sh "mvn clean install jacoco:report -pl ${service} -am"

    junit testResults: '**/target/surefire-reports/*.xml', skipPublishingChecks: true
    processCoverage(service)

    echo "Running SonarQube analysis for ${service}..."
    runBackendSonarQube(service)
    timeout(time: 5, unit: 'MINUTES') {
        waitForQualityGate abortPipeline: true
    }

    echo "Scanning backend dependencies for ${service}..."
    runBackendSnyk(service)

    echo "Building and pushing Docker image for ${service}..."
    def shortCommit = env.GIT_COMMIT ? env.GIT_COMMIT.take(8) : getShortCommitHash()
    buildAndPushBackendImage(service, shortCommit, dockerNamespace, dockerCredentialsId)
}

def runFrontendPipelineAndPush(String service, String dockerNamespace, String dockerCredentialsId) {
    runFrontendPipeline(service)
    def shortCommit = env.GIT_COMMIT ? env.GIT_COMMIT.take(8) : getShortCommitHash()
    buildAndPushFrontendImage(service, shortCommit, dockerNamespace, dockerCredentialsId)
}

def getShortCommitHash() {
    return sh(script: 'git rev-parse --short=8 HEAD', returnStdout: true).trim()
}

def buildAndPushBackendImage(String service, String tag, String dockerNamespace, String dockerCredentialsId) {
    withCredentials([usernamePassword(credentialsId: dockerCredentialsId, usernameVariable: 'DOCKERHUB_USERNAME', passwordVariable: 'DOCKERHUB_TOKEN')]) {
        dir(service) {
            def image = "${dockerNamespace}/yas-${service}:${tag}"
            sh """
                echo \"${DOCKERHUB_TOKEN}\" | docker login -u \"${DOCKERHUB_USERNAME}\" --password-stdin
                docker build -t ${image} .
                docker push ${image}
                docker logout
            """
            echo "Pushed image: ${image}"
        }
    }
}

def buildAndPushFrontendImage(String service, String tag, String dockerNamespace, String dockerCredentialsId) {
    def appDirectory = VALID_FRONTEND_APP_NAME[service]
    def imageName = VALID_FRONTEND_IMAGE_NAME[service]
    if (!appDirectory || !imageName) {
        error("Unsupported frontend service: ${service}")
    }

    withCredentials([usernamePassword(credentialsId: dockerCredentialsId, usernameVariable: 'DOCKERHUB_USERNAME', passwordVariable: 'DOCKERHUB_TOKEN')]) {
        dir(appDirectory) {
            def image = "${dockerNamespace}/${imageName}:${tag}"
            sh """
                echo \"${DOCKERHUB_TOKEN}\" | docker login -u \"${DOCKERHUB_USERNAME}\" --password-stdin
                docker build -t ${image} .
                docker push ${image}
                docker logout
            """
            echo "Pushed image: ${image}"
        }
    }
}

def retagAndPushBackendImage(String service, String sourceTag, String targetTag, String dockerNamespace, String dockerCredentialsId) {
    withCredentials([usernamePassword(credentialsId: dockerCredentialsId, usernameVariable: 'DOCKERHUB_USERNAME', passwordVariable: 'DOCKERHUB_TOKEN')]) {
        def sourceImage = "${dockerNamespace}/yas-${service}:${sourceTag}"
        def targetImage = "${dockerNamespace}/yas-${service}:${targetTag}"
        sh """
            echo \"${DOCKERHUB_TOKEN}\" | docker login -u \"${DOCKERHUB_USERNAME}\" --password-stdin
            docker pull ${sourceImage}
            docker tag ${sourceImage} ${targetImage}
            docker push ${targetImage}
            docker logout
        """
        echo "Retagged image: ${sourceImage} -> ${targetImage}"
    }
}

def retagAndPushFrontendImage(String service, String sourceTag, String targetTag, String dockerNamespace, String dockerCredentialsId) {
    def imageName = VALID_FRONTEND_IMAGE_NAME[service]
    if (!imageName) {
        error("Unsupported frontend service: ${service}")
    }

    withCredentials([usernamePassword(credentialsId: dockerCredentialsId, usernameVariable: 'DOCKERHUB_USERNAME', passwordVariable: 'DOCKERHUB_TOKEN')]) {
        def sourceImage = "${dockerNamespace}/${imageName}:${sourceTag}"
        def targetImage = "${dockerNamespace}/${imageName}:${targetTag}"
        sh """
            echo \"${DOCKERHUB_TOKEN}\" | docker login -u \"${DOCKERHUB_USERNAME}\" --password-stdin
            docker pull ${sourceImage}
            docker tag ${sourceImage} ${targetImage}
            docker push ${targetImage}
            docker logout
        """
        echo "Retagged image: ${sourceImage} -> ${targetImage}"
    }
}

def writeGitOpsServiceOverride(String environmentName, String service, String tag, String imageRoot) {
    writeFile(
        file: "environments/${environmentName}/services/${service}.yaml",
        text: """\
            ${imageRoot}:
              image:
                tag: "${tag}"
        """.stripIndent()
    )
}

pipeline {
    agent any

    tools {
        maven 'maven3'
        nodejs 'node20'
    }

    environment {
        // Use local repository within the workspace for faster caching
        MAVEN_OPTS = "-Dmaven.repo.local=.m2/repository"
        // Required for Testcontainers to communicate with the Docker daemon inside Jenkins agents
        TESTCONTAINERS_HOST_OVERRIDE = 'docker'
        // Explicitly set Docker API version to ensure compatibility with older Docker versions
        DOCKER_API_VERSION = '1.43'

        // Docker and GitOps configuration
        DOCKERHUB_NAMESPACE = 'phatnguyen9725'
        DOCKERHUB_CREDENTIALS_ID = 'dockerhub-token'
        GITOPS_REPO_URL = 'https://github.com/phatnguyen975/GitOps-YAS.git'
        GITOPS_DIR = 'gitops-yas'
        GITOPS_TOKEN_CREDENTIALS_ID = 'gitops-token'
        GITOPS_COMMIT_USER = 'jenkins-bot'
        GITOPS_COMMIT_EMAIL = 'jenkins@local'
    }

    stages {
        // --- STAGE 1: CHECKOUT CODE ---
        stage('Checkout Code') {
            steps {
                checkout scm
                script {
                    echo "Checking out branch: ${env.BRANCH_NAME}"
                }
            }
        }

        // --- STAGE 2: SECRET SCAN ---
        // Scans the repository for accidentally committed secrets/credentials
        stage('Secret Scan') {
            steps {
                script {
                    echo "Checking for secrets..."
                    if (!fileExists('gitleaks')) {
                        echo "Downloading Gitleaks..."
                        sh 'curl -ssfL https://github.com/gitleaks/gitleaks/releases/download/v8.18.2/gitleaks_8.18.2_linux_x64.tar.gz | tar -xz gitleaks'
                    }
                    sh 'chmod +x gitleaks'

                    try {
                        sh './gitleaks detect --source . --config gitleaks.toml --verbose --no-git'
                    } catch (Exception e) {
                        currentBuild.result = 'FAILURE'
                        error("Gitleaks found secrets in your code")
                    }
                }
            }
        }

        // --- STAGE 3: ANALYZE CHANGES ---
        // Determines exactly which services in the monorepo were modified in this commit/PR
        stage('Analyze Changes') {
            steps {
                script {
                    echo "Analyzing changes to determine build scope..."

                    def baseBranch = env.CHANGE_TARGET ?: 'main'
                    def diffCommand = ""

                    if (env.BRANCH_NAME == 'main') {
                        def hasParent = sh(script: "git rev-parse HEAD~1", returnStatus: true) == 0
                        if (hasParent) {
                            diffCommand = "git diff --name-only HEAD~1 HEAD"
                        } else {
                            diffCommand = "git show --name-only --pretty='' HEAD"
                        }
                    } else {
                        sh "git fetch origin ${baseBranch}:refs/remotes/origin/${baseBranch} --depth=10"
                        diffCommand = "git diff --name-only origin/${baseBranch} HEAD"
                    }

                    def changedFilesList = []
                    try {
                        def changedFilesRaw = sh(script: diffCommand, returnStdout: true).trim()
                        changedFilesList = changedFilesRaw ? changedFilesRaw.readLines().findAll { it?.trim() } : []
                    } catch (Exception e) {
                        echo "First build or error detected. Need to build ALL."
                        IS_ROOT_CHANGED = true
                    }
                    echo "List of changed files:\n${changedFilesList.join('\n')}"

                    def servicesToBuild = [] as LinkedHashSet

                    for (String file in changedFilesList) {
                        if (!file) continue

                        if (file == 'pom.xml') {
                            IS_ROOT_CHANGED = true
                        }

                        if (file.startsWith('common-library/')) {
                            echo 'Common library changed. Marking all valid backend services...'
                            servicesToBuild.addAll(VALID_BACKEND_SERVICES)
                        }

                        if (file.startsWith('backoffice/')) BUILD_BACKOFFICE = true
                        if (file.startsWith('storefront/')) BUILD_STOREFRONT = true

                        def topLevelDir = file.tokenize('/').first()
                        if (topLevelDir) {
                            if (VALID_BACKEND_SERVICES.contains(topLevelDir)) {
                                servicesToBuild.add(topLevelDir)
                            }
                        }
                    }

                    if (IS_ROOT_CHANGED) {
                        echo 'Root configuration changed. Building ALL services.'
                        BUILD_BACKOFFICE = true
                        BUILD_STOREFRONT = true
                        CHANGED_SERVICES = ''
                    } else {
                        CHANGED_SERVICES = servicesToBuild.join(',')
                    }

                    echo '---------- BUILD PLAN ----------'
                    echo "Backend Services: ${IS_ROOT_CHANGED ? 'ALL' : (CHANGED_SERVICES ?: 'N/A')}"
                    echo "Frontend Backoffice: ${BUILD_BACKOFFICE}"
                    echo "Frontend Storefront: ${BUILD_STOREFRONT}"
                    echo '--------------------------------'
                }
            }
        }

        // --- STAGE 4: DYNAMIC PARALLEL INTEGRATION & VALIDATION ---
        // Dynamically provisions isolated Jenkins executors to run builds in parallel
        stage('Integration & Validation') {
            steps {
                script {
                    def parallelBranches = [:]
                    def backendServices = resolveBackendServices(IS_ROOT_CHANGED, CHANGED_SERVICES)
                    def frontendServices = resolveFrontendServices(IS_ROOT_CHANGED, BUILD_BACKOFFICE, BUILD_STOREFRONT)

                    def executionPlan = []
                    backendServices.each { executionPlan << [type: 'backend', service: it] }
                    frontendServices.each { executionPlan << [type: 'frontend', service: it] }

                    executionPlan.each { planItem ->
                        def pipelineType = planItem.type
                        def currentService = planItem.service

                        parallelBranches["${pipelineType.capitalize()}-${currentService}"] = {
                            node() {
                                stage("Pipeline: ${currentService}") {
                                    checkout scm

                                    if (pipelineType == 'backend') {
                                        runBackendPipelineAndPush(currentService, env.DOCKERHUB_NAMESPACE, env.DOCKERHUB_CREDENTIALS_ID)
                                        cleanupLocalM2Repo(3)
                                    } else {
                                        runFrontendPipelineAndPush(currentService, env.DOCKERHUB_NAMESPACE, env.DOCKERHUB_CREDENTIALS_ID)
                                    }

                                    cleanWs()
                                }
                            }
                        }
                    }

                    if (parallelBranches.size() > 0) {
                        echo "Executing ${parallelBranches.size()} pipelines in parallel..."
                        parallel parallelBranches
                    } else {
                        echo 'No services affected. Skipping Validation stage.'
                    }
                }
            }
        }

        // --- STAGE 5: MAIN -> DEV GITOPS ---
        stage('CD Dev GitOps Update') {
            when {
                branch 'main'
            }
            steps {
                script {
                    def backendServices = resolveBackendServices(IS_ROOT_CHANGED, CHANGED_SERVICES)
                    def frontendServices = resolveFrontendServices(IS_ROOT_CHANGED, BUILD_BACKOFFICE, BUILD_STOREFRONT)

                    if (backendServices.isEmpty() && frontendServices.isEmpty()) {
                        echo 'No service changes to promote to dev. Skipping stage.'
                        return
                    }

                    checkout scm
                    def shortCommit = env.GIT_COMMIT ? env.GIT_COMMIT.take(8) : getShortCommitHash()

                    // Dual tagging: commit-id and main
                    backendServices.each { String service ->
                        retagAndPushBackendImage(service, shortCommit, 'main', env.DOCKERHUB_NAMESPACE, env.DOCKERHUB_CREDENTIALS_ID)
                    }
                    frontendServices.each { String service ->
                        retagAndPushFrontendImage(service, shortCommit, 'main', env.DOCKERHUB_NAMESPACE, env.DOCKERHUB_CREDENTIALS_ID)
                    }

                    // Update GitOps dev service overrides
                    withCredentials([string(credentialsId: env.GITOPS_TOKEN_CREDENTIALS_ID, variable: 'GITOPS_TOKEN')]) {
                        def repoNoProtocol = env.GITOPS_REPO_URL.replaceFirst('https://', '')
                        sh """
                            rm -rf ${env.GITOPS_DIR}
                            git clone https://x-access-token:\${GITOPS_TOKEN}@${repoNoProtocol} ${env.GITOPS_DIR}
                        """
                    }

                    dir("${env.GITOPS_DIR}") {
                        backendServices.each { String service ->
                            writeGitOpsServiceOverride('dev', service, shortCommit, 'backend')
                        }
                        frontendServices.each { String service ->
                            writeGitOpsServiceOverride('dev', service, shortCommit, 'ui')
                        }

                        def promotedServices = backendServices + frontendServices

                        sh """
                            git config user.name \"${env.GITOPS_COMMIT_USER}\"
                            git config user.email \"${env.GITOPS_COMMIT_EMAIL}\"
                            git add environments/dev/services
                            if git diff --cached --quiet; then
                                echo \"No dev GitOps changes to commit.\"
                            else
                                git commit -m \"ci(dev): update ${shortCommit} for ${promotedServices.join(',')}\"
                                git push origin HEAD:main
                            fi
                        """
                    }
                }
            }
        }

        // --- STAGE 6: MAIN -> STAGING GITOPS ---
        stage('CD Staging GitOps Update') {
            when {
                tag 'v*'
            }
            steps {
                script {
                    def releaseTag = env.TAG_NAME ?: sh(script: 'git describe --tags --exact-match', returnStdout: true).trim()
                    def backendServices = VALID_BACKEND_SERVICES
                    def frontendServices = VALID_FRONTEND_APP_NAME.keySet() as List

                    // Prepare release tag for all services using the latest main image as source
                    backendServices.each { String service ->
                        retagAndPushBackendImage(service, 'main', releaseTag, env.DOCKERHUB_NAMESPACE, env.DOCKERHUB_CREDENTIALS_ID)
                    }
                    frontendServices.each { String service ->
                        retagAndPushFrontendImage(service, 'main', releaseTag, env.DOCKERHUB_NAMESPACE, env.DOCKERHUB_CREDENTIALS_ID)
                    }

                    // Update all staging service overrides
                    withCredentials([string(credentialsId: env.GITOPS_TOKEN_CREDENTIALS_ID, variable: 'GITOPS_TOKEN')]) {
                        def repoNoProtocol = env.GITOPS_REPO_URL.replaceFirst('https://', '')
                        sh """
                            rm -rf ${env.GITOPS_DIR}
                            git clone https://x-access-token:\${GITOPS_TOKEN}@${repoNoProtocol} ${env.GITOPS_DIR}
                        """
                    }

                    dir("${env.GITOPS_DIR}") {
                        backendServices.each { String service ->
                            writeGitOpsServiceOverride('staging', service, releaseTag, 'backend')
                        }
                        frontendServices.each { String service ->
                            writeGitOpsServiceOverride('staging', service, releaseTag, 'ui')
                        }

                        sh """
                            git config user.name \"${env.GITOPS_COMMIT_USER}\"
                            git config user.email \"${env.GITOPS_COMMIT_EMAIL}\"
                            git add environments/staging/services
                            if git diff --cached --quiet; then
                                echo \"No staging GitOps changes to commit.\"
                            else
                                git commit -m \"ci(staging): release ${releaseTag}\"
                                git push origin HEAD:main
                            fi
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                cleanupLocalM2Repo(3)
            }
            sh 'rm -f gitleaks'
            cleanWs()
        }
        success {
            echo '[SUCCESS] CI Pipeline completed successfully!'
        }
        failure {
            echo '[FAILURE] CI Pipeline failed! Check logs for compile errors, test failures, or security issues.'
        }
    }
}
