<p align="center">
  <a href="https://github.com/lupaxa-docker-toolbox">
    <img src="https://raw.githubusercontent.com/the-lupaxa-project/brand-assets/master/logos/organisations/docker-toolbox/readme-logo.png" alt="The Lupaxa Project: Docker Toolbox" />
  </a>
</p>

<h1 align="center">Version Helper</h1>

Version Helper prints a pinned package install fragment for a Dockerfile. It starts a distro image, asks that image's package manager for the current version of each package, and writes install lines you can place under a `RUN` instruction.

[Container Framework](https://github.com/lupaxa-docker-toolbox/container-framework) downloads these scripts when it generates Dockerfiles.

## How it Works

`get-versions` runs `version-grabber` inside the image you name:

```shell
docker run --rm -v /path/to/version-grabber:/version-grabber:ro --env-file=packages.env alpine:3.24 ash /version-grabber
```

The grabber reads package names from the environment file. stdout is only the fragment. Headers, missing packages, and errors go to stderr. The script exits 1 when it cannot resolve any package.

The script indents the fragment with tabs so it can follow a `RUN \` line. The samples below use spaces. Each block ends with `&& \` so later commands can continue that instruction.

Remove the package lists in your Dockerfile when you want a hadolint cache-cleanup rule satisfied. The pins cover the version rules.

## Supported Operating Systems

The table is every release still in the vendor's own security support. That includes Debian long-term support and Enterprise Linux security maintenance. Paid add-ons such as Ubuntu Pro are outside this list.

Ubuntu entries are the April LTS releases. Each tag below was run with `config/config.example`.

The grabber also runs on an older image when that image is still published and uses apk, apt, dnf, microdnf, yum, tdnf, or pacman.

| Operating System | Tags                           | Image                                                  |
| ---------------- | ------------------------------ | ------------------------------------------------------ |
| Alma Linux       | 8, 9, 10, full and minimal     | [Official Image](https://hub.docker.com/_/almalinux)   |
| Alpine           | 3.22, 3.23, and 3.24           | [Official Image](https://hub.docker.com/_/alpine)      |
| Amazon Linux     | 2023                           | [Official Image](https://hub.docker.com/_/amazonlinux) |
| Arch Linux       | latest                         | [Official Image](https://hub.docker.com/_/archlinux)   |
| CentOS Stream    | stream9, stream10              | [Quay Image](https://quay.io/repository/centos/centos) |
| Debian           | 12, 13, full and slim          | [Official Image](https://hub.docker.com/_/debian)      |
| Oracle Linux     | 8, 9, 10, full and slim        | [Official Image](https://hub.docker.com/_/oraclelinux) |
| Photon           | 4.0 and 5.0                    | [Official Image](https://hub.docker.com/_/photon)      |
| Rocky Linux      | 8, 9, 10, full and minimal     | [Official Image](https://hub.docker.com/_/rockylinux)  |
| Ubuntu           | 22.04, 24.04, and 26.04        | [Official Image](https://hub.docker.com/_/ubuntu)      |

> **Note:** Alpine images have no bash. Pass `-s ash`. Slim images often have no bash. Pass `-s sh`.
>
> Arch Linux publishes `linux/amd64`. On Apple Silicon pass `-P linux/amd64`.
> Pacman's sandbox cannot run under that emulation, so the grabber sets `DisableSandbox` for the query.
> The printed fragment still uses a normal `pacman` command.

## Installation

Docker has to be on `PATH`. Copy both scripts somewhere on `PATH`:

```shell
cp src/get-versions src/version-grabber ~/bin/
chmod +x ~/bin/get-versions ~/bin/version-grabber
```

`get-versions` looks for `version-grabber` beside itself, then in `~/bin`.

## Usage

```shell
get-versions [-h] [-d] [-p] -c config -o image [-g grabber] [-s shell] [-t tag] [-P platform]
  -h, --help       Show the help
  -d, --debug      Trace the host script
  -p, --package    Skip the stderr header
  -c, --config     Package config file
  -g, --grabber    Path to version-grabber
  -o, --os         Image name, for example alpine or ubuntu
  -s, --shell      Shell inside the image (default: bash)
  -t, --tag        Image tag (default: latest)
  -P, --platform   Docker platform, for example linux/amd64
```

Alpine:

```shell
get-versions -c config/config.example -o alpine -t 3.24 -s ash
```

Debian:

```shell
get-versions -c config/config.example -o debian -t 13
```

Arch Linux on Apple Silicon:

```shell
get-versions -c config/config.example -o archlinux -P linux/amd64
```

CentOS Stream:

```shell
get-versions -c config/config.example -o quay.io/centos/centos -t stream10
```

Pull the tag when you want pins from the registry rather than a local image. `docker run` keeps an image that is already present.

## Package Configuration

Package names are a space-separated list with no quotes. `DNF_PACKAGES` falls back to `YUM_PACKAGES` when it is unset, so an older config keeps working.

RPM pins include the release. A non-zero epoch stays in the pin (`openssl-devel-1:3.5.8-2.0.1.el9_8`). An epoch of 0 is omitted.

### Package Manager Lists

```text
APK_PACKAGES=            # Alpine
APK_VIRTUAL_PACKAGES=    # Alpine packages installed without a version
APT_PACKAGES=            # Debian and Ubuntu
PACMAN_PACKAGES=         # Arch Linux
TDNF_PACKAGES=           # Photon
DNF_PACKAGES=            # Alma, Rocky, Oracle, Amazon Linux 2023
YUM_PACKAGES=            # yum, and the fallback for DNF_PACKAGES
YUM_GROUPS=              # quoted group names
DNF_GROUPS=              # quoted group names; falls back to YUM_GROUPS
```

### Operating System Lists

Set `DISCOVER_BY=OS` when one image family needs a different list from another that shares a package manager. The grabber reads `/etc/os-release`.

```text
DISCOVER_BY=OS
ALMA_PACKAGES=
ALMA_GROUPS=
ALPINE_PACKAGES=
ALPINE_VIRTUAL_PACKAGES=
AMAZON_PACKAGES=
AMAZON_GROUPS=
ARCH_PACKAGES=
CENTOS_PACKAGES=
CENTOS_GROUPS=
DEBIAN_PACKAGES=
ORACLE_PACKAGES=
ORACLE_GROUPS=
PHOTON_PACKAGES=
ROCKY_PACKAGES=
ROCKY_GROUPS=
UBUNTU_PACKAGES=
```

### Unversioned Packages and Groups

Alpine meta packages such as `build-base` have no useful pin. List them in `APK_VIRTUAL_PACKAGES`. They are installed with `apk add` and no version.

Groups are a space-separated list of quoted strings:

```text
YUM_GROUPS="Development Tools" "Security Tools"
```

`dnf` and `yum` emit `groupinstall`. A microdnf image installs `dnf` first when a group list is set.

A name with no candidate version is omitted. The grabber reports that name on stderr.

## Sample Output

The same [config/config.example](config/config.example) file produced these fragments.

### Alpine 3.24

```dockerfile
  apk update && \
  apk add --no-cache \
    bash=5.3.9-r1 \
    curl=8.23.0-r0 \
    git=2.54.1-r0 \
    openssl-dev=3.5.9-r0 \
    wget=1.25.0-r3 \
    && \
```

### Debian 13

```dockerfile
  apt-get update && \
  apt-get -y --no-install-recommends install \
    bash=5.2.37-2+b10 \
    curl=8.14.1-2+deb13u7 \
    git=1:2.47.3-0+deb13u1 \
    libssl-dev=3.5.7-1~deb13u3 \
    wget=1.25.0-2 \
    && \
```

### Alma Linux 9

```dockerfile
  dnf makecache && \
  dnf install -y \
    bash-5.1.8-9.el9 \
    curl-7.76.1-43.el9 \
    git-2.52.0-1.el9 \
    openssl-devel-1:3.5.9-1.el9_9 \
    wget-1.21.1-11.el9_8 \
    && \
```

### Oracle Linux 9 Slim

```dockerfile
  microdnf makecache && \
  microdnf install -y \
    bash-5.1.8-9.el9 \
    curl-7.76.1-43.el9 \
    git-2.52.0-1.el9 \
    openssl-devel-1:3.5.8-2.0.1.el9_8 \
    wget-1.21.1-11.el9_8 \
    && \
```

### Amazon Linux 2023

```dockerfile
  dnf makecache && \
  dnf install -y \
    bash-5.2.15-1.amzn2023.0.2 \
    curl-8.21.0-5.amzn2023.0.5 \
    git-2.50.1-1.amzn2023.0.2 \
    openssl-devel-1:3.5.8-1.amzn2023.0.4 \
    wget-1.21.3-1.amzn2023.0.5 \
    && \
```

### Arch Linux

```dockerfile
  pacman -Syu --noconfirm && \
  pacman -S --noconfirm \
    bash=5.3.20-1 \
    curl=8.23.0-2 \
    git=2.56.1-1 \
    openssl=4.1.0-1 \
    wget=1.25.0-6 \
    && \
```

### Photon 5.0

```dockerfile
  tdnf makecache && \
  tdnf install -y \
    bash-5.3-1.ph5 \
    curl-8.23.0-1.ph5 \
    git-2.53.0-2.ph5 \
    openssl-devel-3.5.9-3.ph5 \
    wget-1.21.3-10.ph5 \
    && \
```

### Ubuntu Versions

**Ubuntu 22.04**

```dockerfile
  apt-get update && \
  apt-get -y --no-install-recommends install \
    bash=5.1-6ubuntu1.1 \
    curl=7.81.0-1ubuntu1.30 \
    git=1:2.34.1-1ubuntu1.18 \
    libssl-dev=3.0.2-0ubuntu1.30 \
    wget=1.21.2-2ubuntu1.5 \
    && \
```

**Ubuntu 24.04**

```dockerfile
  apt-get update && \
  apt-get -y --no-install-recommends install \
    bash=5.2.21-2ubuntu4 \
    curl=8.5.0-2ubuntu10.16 \
    git=1:2.43.0-1ubuntu7.4 \
    libssl-dev=3.0.13-0ubuntu3.16 \
    wget=1.21.4-1ubuntu4.5 \
    && \
```

**Ubuntu 26.04**

```dockerfile
  apt-get update && \
  apt-get -y --no-install-recommends install \
    bash=5.3-2ubuntu1 \
    curl=8.18.0-1ubuntu2.8 \
    git=1:2.53.0-1ubuntu1.1 \
    libssl-dev=3.5.5-1ubuntu3.7 \
    wget=1.25.0-2ubuntu4.4 \
    && \
```

## Config File

[config/config.example](config/config.example):

```text
APK_PACKAGES=bash curl git openssl-dev wget
APK_VIRTUAL_PACKAGES=
APT_PACKAGES=bash curl git libssl-dev wget
PACMAN_PACKAGES=bash curl git openssl wget
TDNF_PACKAGES=bash curl git openssl-devel wget
DNF_PACKAGES=bash curl git openssl-devel wget
YUM_PACKAGES=bash curl git openssl-devel wget
YUM_GROUPS=
```

## Caveats

Images that share a package manager still disagree about which packages are already installed. A list that is the union of those images is the practical fix. An already installed package does not change the image.

<a href="https://github.com/the-lupaxa-project">
    <img src="https://raw.githubusercontent.com/the-lupaxa-project/brand-assets/master/logos/components/footer-for-child-orgs.svg" alt="The Lupaxa Project Footer" width="100%" />
</a>
