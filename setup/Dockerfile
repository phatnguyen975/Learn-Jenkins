FROM jenkins/jenkins:2.541.2-jdk21

# Switch to root to install system dependencies
USER root

# Install Docker CLI
RUN apt-get update && apt-get install -y lsb-release
RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
    https://download.docker.com/linux/debian/gpg
RUN echo "deb [arch=$(dpkg --print-architecture) \
    signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
    https://download.docker.com/linux/debian \
    $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
RUN apt-get update && apt-get install -y docker-ce-cli

# Install kubectl
RUN curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" \
    && chmod +x ./kubectl \
    && mv ./kubectl /usr/local/bin/kubectl

# Install Helm
RUN curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4 \
    && chmod 700 get_helm.sh \
    && ./get_helm.sh \
    && rm get_helm.sh

# Switch back to jenkins user
USER jenkins

# Copy the plugin list from your host to the container image
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt

# Install all plugins defined in plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt
