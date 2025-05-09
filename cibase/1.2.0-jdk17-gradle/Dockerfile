FROM  mikefarah/yq:4.35.1 AS yq
FROM  docker.m.daocloud.io/library/docker:23.0.6-cli AS docker
FROM gcr.io/kaniko-project/executor:v1.23.2 AS kaniko

FROM docker.m.daocloud.io/library/gradle:8.11-jdk17-jammy
# disable 'maven-default-http-blocker'
#RUN sed -i '160,166d' /usr/share/maven/conf/settings.xml

ENV TZ="Asia/Shanghai" \
    ROOT_PASSWORD="changeit" \
    HELM_VERSION="v3.12.3" \
    PATH="/kaniko:${PATH}"

# install yq
COPY --from=yq /usr/bin/yq /usr/bin/yq

# install kaniko
COPY --from=kaniko /kaniko /kaniko
COPY --from=kaniko /etc/nsswitch.conf /etc/nsswitch.conf

# install docker-client-cli
COPY --from=docker /usr/local/bin/docker /usr/local/bin/docker
COPY --from=docker /usr/local/libexec/docker/cli-plugins/docker-buildx /usr/local/libexec/docker/cli-plugins/docker-buildx

# install base packages
RUN set -eux; \
    apt-get update; \
    DEBIAN_FRONTEND=noninteractive \
    apt-get install -y \
        jq \
        vim \
        git \
        tar \
        curl \
        wget \
        unzip \
        pylint \
        gnupg2 \
        skopeo \
        xmlstarlet \
        openssh-server \
        mariadb-client \
        ca-certificates \
        build-essential \
        apt-transport-https; \
	rm -rf /var/lib/apt/lists/*; \
    ARCH="$(dpkg --print-architecture)"; \
    # install helm
    wget -qO "/tmp/helm-${HELM_VERSION}-linux-${ARCH}.tar.gz" \
        "https://get.helm.sh/helm-${HELM_VERSION}-linux-${ARCH}.tar.gz"; \
    tar xzf "/tmp/helm-${HELM_VERSION}-linux-${ARCH}.tar.gz" -C /tmp; \
    mv /tmp/linux-${ARCH}/helm /usr/bin/helm; \
    # post install
    helm plugin install https://github.com/chartmuseum/helm-push; \
    ln -s /kaniko/executor /kaniko/kaniko; \
    ln -s /usr/bin/xmlstarlet /usr/bin/xml; \
    docker-credential-gcr config --token-source=env; \
    # Modify `sshd_config`
    sed -ri 's/^#PermitEmptyPasswords no/PermitEmptyPasswords yes/' /etc/ssh/sshd_config; \
    sed -ri 's/^#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config; \
    sed -ri 's/^UsePAM yes/UsePAM no/' /etc/ssh/sshd_config; \
    # Delete root password (set as empty)
    passwd -d root; \
    mkdir -p /run/sshd; \
    rm -r /tmp/*;

# Add minio client
RUN curl https://dl.min.io/client/mc/release/linux-ppc64le/mc -o /usr/local/bin/mc; \
    chmod +x /usr/local/bin/mc

# Add trivy
RUN curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin v0.44.1; \
    trivy image --download-db-only;\
    trivy image --download-java-db-only;

# Add mirror source
RUN sed -i "s@http://.*archive.ubuntu.com@http://repo.huaweicloud.com@g" /etc/apt/sources.list; \
    sed -i "s@http://.*security.ubuntu.com@http://repo.huaweicloud.com@g" /etc/apt/sources.list

EXPOSE 22

CMD [ "/bin/sh","-c","/usr/sbin/sshd -D" ]