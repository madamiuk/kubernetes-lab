# 01 — Kubernetes Lab Environment Setup

## 1. Objective

Prepare a lightweight, ARM64-compatible container environment for running local Kubernetes clusters using kind.

The environment should avoid Docker Desktop and additional UTM virtual machines, minimizing the resource footprint on the macOS host.

**Related laboratories:** K8S-001, K8S-002, K8S-003, K8S-004.

## 2. Host Environment

| Component | Value |
|---|---|
| Hardware | Apple Silicon MacBook M2 |
| Operating System | macOS 26.7.1 |
| Architecture | arm64 |
| Homebrew | 7.0.11 |
| Physical memory | 16 GiB |
| Logical CPUs | 10 |
| Physical CPUs reported | 10 |
| Available disk space at initial check | 155 GiB |

### Verification commands

```bash
uname -m
sw_vers
brew --version

sysctl -n hw.memsize
sysctl -n hw.logicalcpu
sysctl -n hw.physicalcpu

df -h /
```

**Result:** Hardware resources were sufficient to run a local Kubernetes learning environment.

## 3. Tool Installation

The initial environment had no installed Kubernetes or container management tools.

The following software was installed using Homebrew:

```bash
brew install colima docker kubectl kind
```

### Installed versions

| Software | Version |
|---|---|
| Colima | 0.10.3 |
| Docker CLI | 29.9.2 |
| kubectl | 1.37.2 |
| kind | 0.33.0 |

### Verification

```bash
colima version
docker --version
kubectl version --client
kind version

brew list --versions colima docker kubernetes-cli kind
```

**Result:** All required tools installed successfully.

## 4. Colima Configuration

Colima was selected as a lightweight Linux virtual machine environment for running the Docker Engine.

The following resources were allocated:

| Resource | Allocation |
|---|---|
| Virtual CPUs | 2 |
| Memory | 4 GiB |
| Disk | 30 GiB |
| Virtualization backend | VZ |
| Container runtime | Docker |
| Kubernetes built into Colima | Disabled |

### Startup command

```bash
colima start \
  --cpus 2 \
  --memory 4 \
  --disk 30 \
  --vm-type vz \
  --runtime docker
```

### Verification commands

```bash
colima status
colima list

docker context ls
docker version

docker info --format \
  'CPUs: {{.NCPU}} | Memory: {{.MemTotal}} bytes'
```

### Observed results

```text
PROFILE   STATUS    ARCH      CPUS   MEMORY   DISK    RUNTIME
default   Running   aarch64   2      4GiB     30GiB   docker
```

The active Docker context was `colima`.

Docker CLI 29.9.2 communicated successfully with Docker Engine 29.5.2 running on Linux ARM64.

The Docker API version was negotiated to 1.54, compatible with the server.

## 5. Troubleshooting — Missing Docker Credential Helper

### Problem

The initial Docker image test failed:

```bash
docker run --rm hello-world
```

Error:

```text
docker: error getting credentials -
err: exec: "docker-credential-desktop":
executable file not found in $PATH
```

### Root cause

The Docker CLI configuration contained a stale credential-helper setting left behind after Docker Desktop had been removed.

The file was located at:

```text
~/.docker/config.json
```

Diagnostic checks found:

```text
credsStore: desktop
credHelpers: {}
currentContext: colima
```

The executable `docker-credential-desktop` was no longer present.

### Resolution

First, a backup of the configuration was created:

```bash
cp -p ~/.docker/config.json ~/.docker/config.json.bak
chmod 600 ~/.docker/config.json.bak
```

Then the configuration was edited:

```bash
vim ~/.docker/config.json
```

The obsolete setting was removed:

```json
"credsStore": "desktop"
```

The active Docker context `colima` was retained.

### Configuration verification

```bash
python3 -m json.tool ~/.docker/config.json >/dev/null \
  && echo "JSON syntax: OK"
```

### Functional verification

```bash
docker run --rm hello-world
```

The Docker client successfully downloaded the ARM64 image and started the container.

Observed result:

```text
Hello from Docker!

This message shows that your installation appears
to be working correctly.
```

**Conclusion:** The issue was caused by a stale Docker CLI credential-helper reference, not a failure of Colima or Docker Engine.

## 6. Environment Architecture

```text
Apple Silicon MacBook
|
+-- macOS ARM64
    |
    +-- Homebrew
    |   +-- kubectl
    |   +-- kind
    |   +-- Docker CLI
    |   +-- Colima
    |
    +-- Colima VM
        |
        +-- Linux ARM64
        +-- 2 vCPU
        +-- 4 GiB RAM
        +-- 30 GiB disk
        |
        +-- Docker Engine
            |
            +-- kind Kubernetes nodes
```

The macOS Docker client communicates with Docker Engine through the active Colima Docker context.

Kubernetes clusters are created independently with kind rather than using Colima's integrated Kubernetes option.

## 7. Useful Administration Commands

Check Colima status:

```bash
colima status
```

Show Docker context:

```bash
docker context show
```

Inspect running containers:

```bash
docker ps
```

Stop Colima when the laboratory is not in use:

```bash
colima stop
```

Restart the existing Colima environment:

```bash
colima start
```

**Important:** Stopping Colima interrupts the local container environment, including kind clusters. It does not intentionally delete the existing cluster configuration or persistent virtual disk.

## 8. Lessons Learned

1. Docker CLI and Docker Engine are separate components.
2. Colima provides a Linux runtime for containers on macOS.
3. Docker contexts determine which Docker Engine receives commands.
4. Old credential-helper configurations may remain after software removal.
5. ARM64-compatible images are important on Apple Silicon.
6. Resource allocation should be planned before creating local Kubernetes clusters.
7. The Docker Engine must be operational before kind can create a Kubernetes cluster.

## 9. Laboratory Result

**Status: Completed**

The macOS environment is prepared for running local Kubernetes clusters using kind.

The Docker Engine and Docker CLI were verified through a successful `hello-world` container execution.
