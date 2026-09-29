# ==============================================================================
# Jenkins controller with the Docker CLI + compose plugin installed, so the
# Jenkinsfile's `docker build` / `docker push` / `docker compose up` steps run
# against the HOST's Docker engine via the mounted socket (docker-outside-of-
# docker). This is a local/dev CI box, not a hardened multi-tenant controller.
# ==============================================================================
FROM jenkins/jenkins:lts

USER root

RUN apt-get update && apt-get install -y --no-install-recommends \
        ca-certificates curl gnupg \
    && install -m 0755 -d /etc/apt/keyrings \
    && curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg \
    && chmod a+r /etc/apt/keyrings/docker.gpg \
    && . /etc/os-release \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian ${VERSION_CODENAME} stable" \
        > /etc/apt/sources.list.d/docker.list \
    && apt-get update && apt-get install -y --no-install-recommends \
        docker-ce-cli docker-compose-plugin \
    && rm -rf /var/lib/apt/lists/*

# Jenkins runs as root here specifically so it can read the mounted
# /var/run/docker.sock, whose group ownership belongs to the host and can't
# be predicted at image-build time.
USER root
