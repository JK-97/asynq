# Use the official Go image as base
FROM mcr.microsoft.com/devcontainers/go:1-1.22-bookworm

# Set Go proxy for faster module downloads
ENV GOPROXY=https://goproxy.cn,https://goproxy.io,https://proxy.golang.org,direct \
    GOSUMDB=sum.golang.org \
    GO111MODULE=on

# Set the working directory
WORKDIR /workspace

# Note: Go tools will be installed via postCreateCommand to avoid build-time issues
