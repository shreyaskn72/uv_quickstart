FROM mcr.microsoft.com/devcontainers/base:ubuntu

ARG KUBECTL_VERSION=v1.37.0
ARG KIND_VERSION=v0.30.0

RUN curl -fsSL -o /tmp/kubectl "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl" \
    && install -o root -g root -m 0755 /tmp/kubectl /usr/local/bin/kubectl \
    && rm /tmp/kubectl \
    && curl -fsSL -o /usr/local/bin/kind "https://kind.sigs.k8s.io/dl/${KIND_VERSION}/kind-linux-amd64" \
    && chmod +x /usr/local/bin/kind \
    && curl -fsSL "https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4" | bash \
    && helm version --short \
    && kubectl version --client \
    && kind version