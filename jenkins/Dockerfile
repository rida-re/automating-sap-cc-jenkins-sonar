FROM jenkins/jenkins:lts-jdk17

USER root

# -------------------------
# Install Node.js 22 + Angular CLI
# -------------------------
RUN curl -fsSL https://deb.nodesource.com/setup_22.x | bash - \
    && apt-get install -y nodejs \
    && npm install -g @angular/cli \
    && node -v && npm -v

# -------------------------
# Install Jenkins Plugins
# -------------------------
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt

USER jenkins
