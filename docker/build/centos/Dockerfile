FROM centos:centos7.9.2009

# Install OS dependencies to accomplish the following:
#     1. node-native builds
#     2. headless browser testing
#     3. managing git installs
#     4. general development utilities
RUN yum update -y && \
    yum install -y \
        libXext-devel-0:1.3.3-3.el7.x86_64 \
        libXi-devel-0:1.7.9-1.el7.x86_64 \
        mesa-libGL-devel-0:18.3.4-12.el7_9.x86_64 \
        mesa-dri-drivers-0:18.3.4-12.el7_9.x86_64 \
        xorg-x11-server-Xvfb-0:1.20.4-17.el7_9.x86_64 \
        wget-0:1.14-18.el7_6.1.x86_64 \
        which-0:2.20-7.el7.x86_64 \
        make-1:3.82-24.el7.x86_64 \
        scl-utils-0:20130529-19.el7.x86_64 \
        centos-release-scl-0:2-3.el7.centos.noarch \
        git-0:1.8.3.1-23.el7_8.x86_64 \
        java-1.8.0-openjdk-headless-1:1.8.0.322.b06-1.el7_9.x86_64 \
        bzip2-0:1.0.6-13.el7.x86_64 && \
    yum clean all

# These must be installed after scl-utils so that the gpg key is available
RUN yum install -y \
        devtoolset-8-gcc-0:8.3.1-3.2.el7.x86_64 \
        devtoolset-8-gcc-c++-0:8.3.1-3.2.el7.x86_64 && \
    yum clean all

# Install nvm with node and npm for both app and test
# since they run on different versions of node.
ENV NVM_DIR /usr/local/nvm
ENV APP_NODE_VERSION 12.21.0
ENV APP_NPM_VERSION 6.14.10
ENV TEST_NODE_VERSION 8.11.4
ENV TEST_NPM_VERSION 6.4.1
RUN mkdir /usr/local/nvm
RUN curl -sL https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash && \
    . $NVM_DIR/nvm.sh && \
    nvm install $TEST_NODE_VERSION && \
    nvm install $APP_NODE_VERSION && \
    nvm alias default $APP_NODE_VERSION && \
    nvm use default
# Default to app configuration, override for test containers
ENV NODE_PATH $NVM_DIR/v$APP_NODE_VERSION/lib/node_modules
ENV PATH      $NVM_DIR/versions/node/v$APP_NODE_VERSION/bin:$PATH

# Configure git for git installs for both npm and pip
RUN git config --system url."https://github.com".insteadOf git@github.com:
RUN git config --system url."https://".insteadOf git://

# Install a chrome binary for "headful"-headless testing
RUN wget -q https://dl.google.com/linux/direct/google-chrome-stable_current_x86_64.rpm && \
    yum localinstall -y google-chrome-stable_current_x86_64.rpm && \
    yum clean all

