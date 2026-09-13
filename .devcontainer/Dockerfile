FROM mcr.microsoft.com/devcontainers/base:ubuntu-22.04

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        gcc-multilib \
        libc6-dev-i386 \
        binutils \
        make \
        file \
        gdb \
        openjdk-17-jdk-headless \
        ant \
        javacc \
    && rm -rf /var/lib/apt/lists/*
