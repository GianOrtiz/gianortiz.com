+++
date = '2025-03-18T18:12:50-03:00'
draft = false
title = 'Kind with CRIU'
+++

While developing an operator over the Checkpoint feature of Kubernetes I needed a setup to run locally to test the operator. Kind is a tool for running local Kubernetes clusters using Docker containers. Unfortunately, Kind does not support the Checkpoint feature natively. So, while searching for a solution I found out [Use CRI-O Container Runtime with Kind](https://rkiselenko.dev/blog/crio-in-kind/), so I have used this tutorial as a starting point in running a Kind cluster with Checkpoint feature.

## How Kubernetes Checkpoint feature works

The Kubernetes Checkpoint feature is implemented into the kubelet. It uses the CRI (Container Runtime Interface) to checkpoint a Pod container. The implementation of each CRI is different, but, the CRIs that implement the Checkpoint feature use the CRIU (Checkpoint Restore in Userspace) tool to checkpoint the containers. To create a Kind cluster with Checkpoint feature, we need to install CRIU in the container image that Kind runs.

So, we will need to create a custom Kind image with CRIU installed and a CRI that supports the Checkpoint feature, which CRI-O does.

## Creating a custom Kind image with CRIU installed

First, we need to understand the steps performed at [Use CRI-O Container Runtime with Kind](https://rkiselenko.dev/blog/crio-in-kind/). I have condensed the steps into this bash script so I can run it when I need it:

```bash
#!/bin/bash

usage() {
    echo "Fail"
    echo "Usage: K8S_VERSION=v1.30.0 CRIO_VERSION=v1.30 ./scripts/build-node-image.sh"
    exit 1
}

if [ -z "$K8S_VERSION" ]; then
    usage
fi

if [ -z "$CRIO_VERSION" ]; then
    usage
fi

# Build the base image.
KIND_DIR=$(mktemp -d)
git clone git@github.com:kubernetes-sigs/kind.git "$KIND_DIR"
cd "$KIND_DIR"/images/base
# Extract the base image tag from make quick output
BASE_IMAGE=$(make quick 2>&1 | grep "docker buildx build" | grep -o "gcr.io/k8s-staging-kind/base:[^ ]*")
if [ -z "$BASE_IMAGE" ]; then
    echo "Failed to extract base image tag from make quick output"
    exit 1
fi
cd -

rm -rf "$KIND_DIR"

# Clone kubernetes repository in order for kind to work while building node image.
KUBERNETES_PATH="$GOPATH/src/k8s.io/kubernetes"
if [ -d "$KUBERNETES_PATH" ]; then
    rm -rf "$KUBERNETES_PATH"
fi
mkdir -p "$KUBERNETES_PATH"
git clone --depth 1 --branch ${K8S_VERSION} https://github.com/kubernetes/kubernetes.git "$KUBERNETES_PATH"

# Build the node image.
kind build node-image --base-image ${BASE_IMAGE}

# Build the final kind image with CRIU.
docker build --build-arg CRIO_VERSION=$CRIO_VERSION -t kindnode/criu:$CRIO_VERSION -f kind-criu.Dockerfile .
```

It clones the Kind repository and builds the base image using the latest version of the Kind base image. Then, it will grep the image generated to use later when building the final node image. Later, we must clone the Kubernetes repository inside the `$GOPATH` for Kind to work while building the node image, it will use the package in the `$GOPATH` in order to build the image. Now, Kind can use the base image generated with `make quick` to build the node image using the packages installed in the Kubernetes repository. The last step is to build the final node image with CRIU installed. Note, that it is important to use the same version of CRI-O and Kubernetes, as CRI-O is specific to Kubernetes. So, if you are running this script you should set the `$K8S_VERSION` to v1.30.0 and `$CRIO_VERSION` to v1.30.

## The customized image

In the last script, we generate the base node image with the command `kind build node-image`, then at the last line we build a custom Dockerfile and use it as the `kindnode/criu` image name, this will be our final node image with CRIU installed. The custom Dockerfile is shown below:

```Dockerfile
FROM kindest/node:latest

ARG CRIO_VERSION
ARG PROJECT_PATH=prerelease:/$CRIO_VERSION

# Install dependencies for CRIU.
RUN apt-get update -y && apt-get install -y \
    build-essential \
    libprotobuf-dev \
    libprotobuf-c-dev \
    protobuf-c-compiler \
    protobuf-compiler \
    python3-protobuf \
    libnl-3-dev \
    libcap-dev \
    libnet-dev \
    pkg-config \
    git \
    wget \
    curl \
    software-properties-common \
    vim \
    gnupg \
    uuid-dev \
    libbsd-dev \
    libdrm-dev \
    gnutls-dev \
    libnftables-dev

# Install CRIU from source so we can use the latest version compatible with the Linux kernel.
RUN cd /tmp && \
    git clone https://github.com/checkpoint-restore/criu.git && \
    cd criu && \
    make && \
    mv criu/criu /usr/bin/criu

# Install cri-o from source using the given version.
RUN echo "Installing Packages ..." \
    && apt-get clean \
    && apt-get update -y \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y \
    software-properties-common vim gnupg \
    && echo "Installing cri-o ..." \
    && curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/$PROJECT_PATH/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg \
    && echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/$PROJECT_PATH/deb/ /" | tee /etc/apt/sources.list.d/cri-o.list \
    && apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get --option=Dpkg::Options::=--force-confdef install -y cri-o \
    && sed -i 's/containerd/crio/g' /etc/crictl.yaml

# Configure cri-o to use CRIU for checkpoint/restore.
COPY crio.conf /etc/crio/crio.conf
COPY crio.conf /etc/crio/crio.conf.d/11-crio.conf

# Configuration so CRIU can checkpoint Pods in the cluster.
COPY criu.conf /etc/criu/default.conf

# Disable containerd and enable cri-o.
RUN systemctl disable containerd && systemctl enable crio
```

This Dockerfile is an adaptation of the Dockerfile used in the [Use CRI-O Container Runtime with Kind](https://rkiselenko.dev/blog/crio-in-kind/) tutorial. Here, we are installing CRIU from the source and configuring both cri-o and CRIU to work together on making checkpoints. We have the following commands to install the required packages for CRIU and build CRIU from source:

```Dockerfile
# Install dependencies for CRIU.
RUN apt-get update -y && apt-get install -y \
    build-essential \
    libprotobuf-dev \
    libprotobuf-c-dev \
    protobuf-c-compiler \
    protobuf-compiler \
    python3-protobuf \
    libnl-3-dev \
    libcap-dev \
    libnet-dev \
    pkg-config \
    git \
    wget \
    curl \
    software-properties-common \
    vim \
    gnupg \
    uuid-dev \
    libbsd-dev \
    libdrm-dev \
    gnutls-dev \
    libnftables-dev

# Install CRIU from source so we can use the latest version compatible with the Linux kernel.
RUN cd /tmp && \
    git clone https://github.com/checkpoint-restore/criu.git && \
    cd criu && \
    make && \
    mv criu/criu /usr/bin/criu
```

The required dependencies for CRIU are installed, then we clone the CRIU repository and build it using `make`. After compiling CRIU, we move it to `/usr/bin/criu` so other programs, like cri-o, can find it. We then install cri-o following the steps in the [Use CRI-O Container Runtime with Kind](https://rkiselenko.dev/blog/crio-in-kind/) tutorial. Lastly, we configure cri-o and CRIU using configuration files:

```Dockerfile
# Configure cri-o to use CRIU for checkpoint/restore.
COPY crio.conf /etc/crio/crio.conf
COPY crio.conf /etc/crio/crio.conf.d/11-crio.conf

# Configuration so CRIU can checkpoint Pods in the cluster.
COPY criu.conf /etc/criu/default.conf

# Disable containerd and enable cri-o.
RUN systemctl disable containerd && systemctl enable crio
```

We set the configuration file for cri-o at `/etc/crio/crio.conf.d/11-crio.conf` so we can override the default configuration. The cri-o configuration is the one below:

```
[crio.runtime]
default_runtime = "runc"
enable_criu_support = true
drop_infra_ctr = false
```

The `default_runtime` configuration will use `runc` as the runtime instead of `crun`, `crun` does not work out well with CRIU in the version we are using as the libcriu2 package is not compiled together with the package. Then, we enable the CRIU support with `enable_criu_support` and remove an infra container from the Pods with `drop_infra_ctr`.

The CRIU configuration file is the one below:

```
tcp-close
skip-in-flight
manage-cgroups=ignore
```

The `skip-in-flight` configuration will skip connections that didn't finish the TCP handshake as we are not interested in this state while checkpointing. The `tcp-close` configuration allows servers to reconnect to the same listening socket after a connection has been closed and established in the restored container. The `manage-cgroups=ignore` configuration will not manage the cgroups of the container when checkpointing.

## Running the cluster

First, we are going to build the node image using the script we created:

```bash
K8S_VERSION=v1.30.0 CRIO_VERSION=v1.30 ./scripts/build-node-image.sh
```

After building the node image, we can run the cluster using the command:

```bash
kind create cluster --image kindnode/criu:v1.30 --config kind-config.yaml
```

The `kind-config.yaml` file is the one below:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
featureGates:
  ContainerCheckpoint:true
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      criSocket: unix:///var/run/crio/crio.sock
- role: worker
  kubeadmConfigPatches:
  - |
    kind: JoinConfiguration
    nodeRegistration:
      criSocket: unix:///var/run/crio/crio.sock
  extraPortMappings:
  - containerPort: 10250
    hostPort: 10250
    protocol: TCP  
```

I like to map port 10250 to the host port 10250 so I can use the kubelet API to interact with the cluster from my local machine while testing.

This should create a cluster that has the Checkpoint feature enabled to start working out with checkpoints of containers locally. Now, you have a consistent environment with CRIU installed in Kubernetes to develop container checkpoint features.