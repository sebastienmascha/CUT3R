# ----- Image with CUDA and cuDNN -----
FROM nvidia/cuda:12.1-cudnn8-devel-ubuntu22.04@sha256:217134b60289ced62b47463be32584329baf35cb55a9ccda6626b91bd59803be AS base-compilation
LABEL maintainer="sebastien@summerrobotics"
LABEL org.opencontainers.image.authors="sebastien@summerrobotics"

ARG TARGETARCH

ARG CUDA_HOME=/usr/local/cuda
ARG PATH=$CUDA_HOME/bin:$PATH
ARG LD_LIBRARY_PATH=$CUDA_HOME/lib64:$LD_LIBRARY_PATH
ENV TORCH_CUDA_ARCH_LIST="7.5;8.0;8.6;8.9"
ENV CMAKE_CUDA_ARCHITECTURES="75;80;86;89"

# ----- APT Configuration -----
# Disable interactive configuration of apt installs
ARG DEBIAN_FRONTEND=noninteractive
# Note: you will still see the following warning once (this can be safely ignored):
# "debconf: delaying package configuration, since apt-utils is not installed"

# Disable NVIDIA PPA for cuda
RUN ls -al /etc/apt && ls -al /etc/apt/sources.list.d && \
    mv /etc/apt/sources.list.d/cuda-ubuntu2204*.list /etc/apt/sources.list.d/cuda-ubuntu2204.list.BAK \
    && rm -rf /var/lib/apt/lists/* && apt-get clean && apt-get autoclean && apt-get -y autoremove
# Clean up steps to minimize image size
# - rm -rf /var/lib/apt/lists/*     Removes the package lists to reduce image size.
# - apt-get clean                   Cleans up the apt cache by removing .deb files.
# - apt-get autoclean               Removes unnecessary package files, further reducing size.
# - apt-get -y autoremove           Removes unneeded packages installed as dependencies.

# ----- Set system timezone: tzdata -----
ENV TZ="Europe/Paris"
RUN apt-get update && apt-get install -y --no-install-recommends \
    apt-utils \
    tzdata \
    && rm -rf /var/lib/apt/lists/* && apt-get clean && apt-get autoclean && apt-get -y autoremove && \
    ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone


# ----- Add APT mirror dependencies -----
RUN apt-get update && apt-get install -y --no-install-recommends \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    software-properties-common \
    tar \
    wget \
    && rm -rf /var/lib/apt/lists/* && apt-get clean && apt-get autoclean && apt-get -y autoremove


# ----- Install compile toolchain: gcc, g++, gfortran (with /test PPA for upstream versions) -----
RUN add-apt-repository ppa:ubuntu-toolchain-r/test && \
    apt-get update && apt-get install -y --no-install-recommends \
    gcc-12 \
    g++-12 \
    gfortran-12 \
    build-essential \
    && mv /etc/apt/sources.list.d/ubuntu-toolchain-r-ubuntu-test-$(lsb_release -sc).list /etc/apt/sources.list.d/ubuntu-toolchain-r-ubuntu-test-$(lsb_release -sc).list.bak && \
    rm -rf /var/lib/apt/lists/* && apt-get clean && apt-get autoclean && apt-get -y autoremove && \
    update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-12 100 --slave /usr/bin/g++ g++ /usr/bin/g++-12 --slave /usr/bin/gcov gcov /usr/bin/gcov-12 --slave /usr/bin/gfortran gfortran /usr/bin/gfortran-12


# ----- Install CMake with Kitware PPA -----
# Using release candidates for Ubuntu. Sources: https://apt.kitware.com
COPY pkgs/compilation_infrastructure/cmake/install_for_linux.sh /tmp/docker/pkgs/compilation_infrastructure/cmake/install_for_linux.sh
RUN bash /tmp/docker/pkgs/compilation_infrastructure/cmake/install_for_linux.sh && rm -rf /tmp/docker

# ----- Setup Python -----
ENV PYTHONFAULTHANDLER=1
# Force stdin, stdout and stderr to be totally unbuffered
ENV PYTHONUNBUFFERED=1
# Don't write .pyc files
ENV PYTHONDONTWRITEBYTECODE=1
ENV PIP_NO_CACHE_DIR=off
# Suppress pip upgrade warning
# PIP_DISABLE_PIP_VERSION_CHECK=on \
ENV PIP_DEFAULT_TIMEOUT=100
ENV VIRTUAL_ENV=/opt/python/venv
ENV PATH="$VIRTUAL_ENV/bin:$PATH"
COPY pkgs/compilation_infrastructure/python3/install_for_linux.sh /tmp/docker/pkgs/compilation_infrastructure/python3/install_for_linux.sh
RUN bash /tmp/docker/pkgs/compilation_infrastructure/python3/install_for_linux.sh && rm -rf /tmp/docker

# -- APT packages for 3rdparty packages in /opt
RUN apt-get update && apt-get install -y --no-install-recommends \
    doxygen \
    git \
    libboost-all-dev=1.74.0.3ubuntu7 \
    ninja-build=1.10.1-1 \
    qtbase5-dev=5.15.3+dfsg-2ubuntu0.2 \
    libqt5widgets5=5.15.3+dfsg-2ubuntu0.2 \
    pkg-config \
    sudo \
    && rm -rf /var/lib/apt/lists/* && apt-get clean && apt-get autoclean && apt-get -y autoremove

# -- Install Conda
# ENV SR_CONDA_ENV_NAME="cut3r"
# ENV SR_CONDA_PATH="/opt/miniconda"
# COPY pkgs/conda/ /tmp/docker/pkgs/conda/
# RUN bash /tmp/docker/pkgs/conda/install_conda.sh $SR_CONDA_PATH $SR_CONDA_ENV_NAME && rm -rf /tmp/docker

# ----- Install PyTorch -----
RUN mkdir -p /opt/cache/python/pip && \
    chmod -R 777 /opt/cache && \
    python3 -m pip install \
        torch==2.6.0 \
        torchvision==0.21.0 \
        --cache-dir /opt/cache/python/pip && \
    chmod -R 777 /opt/python/venv

# Python3
COPY requirements.txt /tmp/requirements.txt
RUN mkdir -p /opt/cache/python/pip && \
    chmod -R 777 /opt/cache && \
    python3 -m pip install -r /tmp/requirements.txt --cache-dir /opt/cache/python/pip && \
    chmod -R 777 /opt/python/venv

# training-logging, evaluation, 3D
RUN mkdir -p /opt/cache/python/pip && \
    chmod -R 777 /opt/cache && \
    python3 -m pip install --no-cache-dir \
      --no-build-isolation \
      git+https://github.com/nerfstudio-project/gsplat.git \
      --cache-dir /opt/cache/python/pip && \
    chmod -R 777 /opt/python/venv

RUN apt-get update && apt-get install -y --no-install-recommends \
    libomp-dev \
    && rm -rf /var/lib/apt/lists/* && apt-get clean && apt-get autoclean && apt-get -y autoremove

COPY src/croco/models/curope /tmp/docker/curope
RUN cd /tmp/docker/curope && \
    python setup.py build_ext --inplace && \
    rm -rf /tmp/docker

# ----- Install ZSH -----
COPY pkgs/dev/zsh/install_for_linux.sh /tmp/docker/pkgs/dev/zsh/install_for_linux.sh
RUN bash /tmp/docker/pkgs/dev/zsh/install_for_linux.sh && rm -rf /tmp/docker
# Configure powerlevel10k theme
COPY pkgs/docker/shell_config/p10k_config_sr.zsh /opt/shell_config/.p10k.zsh

# ----- Environment -----
# Tells GUI programs to communicate with the first local display that was started
ENV DISPLAY=:0
# Remove dbus warning
ENV NO_AT_BRIDGE=1
# Terminal colors
ENV TERM=xterm-256color

# ----- Entrypoint -----
COPY --chmod=755 pkgs/docker/container-entrypoint-base.sh /opt/container/container-entrypoint-base.sh
ENTRYPOINT ["/opt/nvidia/nvidia_entrypoint.sh", "/opt/container/container-entrypoint-base.sh"]