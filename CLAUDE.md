# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

Multi-architecture container for AWS Fargate that provides tools for managing AWS and OpenShift (ROSA) clusters. The container runs with an entrypoint script that supports dynamic version selection via environment variables, defaulting to `sleep infinity`, and is accessed via ECS Exec.

## Building

```bash
# Build both architectures and create manifest list
make all

# Build single architecture
make build-amd64
make build-arm64

# Create manifest list from existing builds
make manifest

# Remove all images and manifests
make clean
```

## Container Architecture

### Multi-Architecture Build System

The Containerfile uses `uname -m` to detect architecture at build time. When podman builds with `--platform linux/arm64`, RUN commands execute in QEMU emulation where `uname -m` returns `aarch64`. For `--platform linux/amd64`, it returns `x86_64`.

- **x86_64 (amd64)**: Uses `x86_64` for AWS CLI, no suffix for OpenShift downloads
- **ARM64 (aarch64)**: Uses `aarch64` for AWS CLI, `-arm64` suffix for OpenShift downloads

Architecture values from `uname -m` are stored in temp files (`/tmp/aws_cli_arch`, `/tmp/oc_suffix`) during build and consumed by subsequent RUN commands.

### Tool Installation via Alternatives System

The container uses Linux alternatives system to manage multiple versions:

**AWS CLI**:
- Fedora RPM (`/usr/bin/aws`) - priority 10, family "fedora"
- Official AWS CLI v2 (`/opt/aws-cli-official/v2/current/bin/aws`) - priority 20, family "aws-official" (default)

**OpenShift CLI**:
- 7 versions installed to `/opt/openshift/{version}/oc`
- Priorities: 14-19 for versions 4.14-4.19, priority 100 for 4.20 (default)
- All downloaded from `https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable-{version}/`

### Critical URLs

- **AWS CLI**: `https://awscli.amazonaws.com/awscli-exe-linux-{arch}.zip`
- **OpenShift CLI**: `https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable-{version}/openshift-client-linux{suffix}.tar.gz`

### Runtime Version Selection

The container includes an entrypoint script (`/usr/local/bin/entrypoint.sh`) that supports dynamic version selection via environment variables:

**Environment Variables**:
- `OC_VERSION`: Select OpenShift CLI version (4.14-4.20, default: 4.20)
- `AWS_CLI`: Select AWS CLI source (`fedora` or `official`, default: official)

**Entrypoint Behavior**:
1. Checks `OC_VERSION` and uses `alternatives --set` to switch to that version if provided
2. Checks `AWS_CLI` and uses `alternatives --set` to switch to fedora/official if provided
3. Executes the command via `exec` (defaults to `sleep infinity`)

The entrypoint is located at `entrypoint.sh` in the repository root and copied to `/usr/local/bin/entrypoint.sh` during build.

## Testing Containers Locally

```bash
# Run with default versions
podman run -it rosa-boundary:latest /bin/bash

# Test with specific OC version
podman run --rm -e OC_VERSION=4.18 rosa-boundary:latest oc version --client

# Test with Fedora AWS CLI
podman run --rm -e AWS_CLI=fedora rosa-boundary:latest aws --version

# Test both environment variables together
podman run -it -e OC_VERSION=4.17 -e AWS_CLI=fedora rosa-boundary:latest /bin/bash

# Verify default tool versions
podman run --rm rosa-boundary:latest sh -c "aws --version && oc version --client"

# Check alternatives configuration (advanced)
podman run --rm rosa-boundary:latest sh -c "alternatives --display aws && alternatives --display oc"
```

## Adding New OpenShift Versions

1. Add version to loop in Containerfile:38 (e.g., `4.21`)
2. Add alternatives registration in Containerfile:47-53 with appropriate priority (e.g., priority 21 for 4.21)
3. Update highest version priority to 100 if it should be the new default
4. Update environment variable documentation in README.md and entrypoint.sh validation

## Manifest List Structure

The `make manifest` target creates an OCI image index containing both architectures. Podman/Docker automatically selects the correct platform when pulling `rosa-boundary:latest`.
