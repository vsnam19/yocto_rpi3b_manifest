# Yocto Linux Distribution for Raspberry Pi 3B

This repository contains a repo manifest for building a custom Linux distribution using the Yocto Project, specifically tailored for the Raspberry Pi 3B.

## Overview

This manifest sets up a complete Yocto build environment with the following components:

- **Base Layer**: Poky (Yocto Project reference distribution)
- **Version**: Scarthgap release
- **Target Hardware**: Raspberry Pi 3B
- **Additional Layers**:
  - meta-openembedded (additional recipes and functionality)
  - meta-security (security-focused recipes)
  - meta-virtualization (containerization and virtualization support)
  - meta-raspberrypi (Raspberry Pi BSP layer)
- **Docker Environment**: Pre-configured Docker environment for consistent builds

## Prerequisites

Before getting started, ensure your build host meets the Yocto Project requirements:

### System Requirements
- **OS**: Ubuntu 22.04 LTS, Fedora 38+, or other supported Linux distributions
- **CPU**: Multi-core processor (quad-core or higher recommended)
- **RAM**: 8GB minimum, 16GB+ recommended
- **Storage**: 100GB+ free disk space
- **Network**: Reliable internet connection for downloading sources

### Required Packages

#### For Ubuntu/Debian:
```bash
sudo apt update
sudo apt install -y gawk wget git diffstat unzip texinfo gcc build-essential \
    chrpath socat cpio python3 python3-pip python3-pexpect xz-utils \
    debianutils iputils-ping python3-git python3-jinja2 libegl1-mesa \
    libsdl1.2-dev pylint3 xterm python3-subunit mesa-common-dev zstd liblz4-tool
```

#### For Fedora/CentOS/RHEL:
```bash
sudo dnf install -y gawk make wget tar bzip2 gzip python3 unzip perl patch \
    diffutils diffstat git cpp gcc gcc-c++ glibc-devel texinfo chrpath \
    ccache perl-Data-Dumper perl-Text-ParseWords perl-Thread-Queue \
    python3-pip xz which SDL-devel xterm rpcgen mesa-libGL-devel \
    perl-FindBin perl-File-Compare perl-File-Copy perl-locale zstd lz4
```

### Install Google Repo Tool
```bash
# Download repo tool
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo

# Add ~/bin to PATH if not already added
echo 'export PATH=~/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

## Quick Start

**Choose your build method:**
- **Native Build**: Follow steps 1-5 below for building directly on your host system
- **Docker Build**: Skip to [Docker Build Option](#docker-build-option) for containerized builds

### 1. Initialize the Build Environment
```bash
# Create workspace directory
mkdir -p ~/yocto-rpi3b
cd ~/yocto-rpi3b

# Initialize repo with this manifest
repo init -u https://github.com/vsnam19/yocto_rpi3b_manifest.git -b main

# Sync all repositories
repo sync
```

### 2. Set Up Build Environment
```bash
# Source the build environment
source poky/oe-init-build-env build-rpi3b

# This will create and enter the build directory
```

### 3. Configure Build
Edit `conf/local.conf` to set your target machine:
```bash
# Set target machine to Raspberry Pi 3B
echo 'MACHINE = "raspberrypi3-64"' >> conf/local.conf

# Optional: Set download directory to save bandwidth on rebuilds
echo 'DL_DIR = "${TOPDIR}/../downloads"' >> conf/local.conf

# Optional: Set shared state cache directory
echo 'SSTATE_DIR = "${TOPDIR}/../sstate-cache"' >> conf/local.conf

# Optional: Enable parallel builds (adjust based on your system)
echo 'BB_NUMBER_THREADS = "8"' >> conf/local.conf
echo 'PARALLEL_MAKE = "-j 8"' >> conf/local.conf
```

Add required layers to `conf/bblayers.conf`:
```bash
# The layers should be automatically configured, but verify they include:
bitbake-layers show-layers
```

### 4. Build the Image
```bash
# Build core minimal image (faster, good for testing)
bitbake core-image-minimal

# OR build a more complete image with development tools
bitbake core-image-full-cmdline

# OR build an image with package management
bitbake core-image-base
```

## Docker Build Option

For a more consistent and isolated build environment, you can use the included Docker setup:

### Prerequisites for Docker Build
- Docker installed and running
- Docker Compose (optional, if provided in docker-env)
- Sufficient disk space for Docker containers and Yocto build

### Using Docker Environment
```bash
# After repo sync, navigate to the docker environment
cd docker-env

# Build and run the Docker container
docker build -t yocto-builder .

# Run container with mounted workspace
docker run -it --rm \
  -v $(pwd)/..:/workspace \
  -v yocto-downloads:/workspace/downloads \
  -v yocto-sstate:/workspace/sstate-cache \
  yocto-builder

# Inside the container, source the build environment
source /workspace/poky/oe-init-build-env /workspace/build-rpi3b

# Configure and build as usual
echo 'MACHINE = "raspberrypi3-64"' >> conf/local.conf
bitbake core-image-minimal
```

### Benefits of Docker Build
- **Consistent Environment**: Same build environment across different host systems
- **Isolation**: No interference with host system packages
- **Reproducibility**: Guaranteed reproducible builds
- **Easy Setup**: No need to install Yocto dependencies on host

### 5. Flash to SD Card
After successful build, the image will be located at:
```
tmp/deploy/images/raspberrypi3-64/
```

Use your preferred method to flash the `.wic.bz2` file to an SD card:

#### Using dd (Linux):
```bash
# Decompress the image
bunzip2 core-image-minimal-raspberrypi3-64.wic.bz2

# Flash to SD card (replace /dev/sdX with your SD card device)
sudo dd if=core-image-minimal-raspberrypi3-64.wic of=/dev/sdX bs=4M status=progress
sudo sync
```

#### Using Raspberry Pi Imager:
1. Download and install [Raspberry Pi Imager](https://rpi.org/imager)
2. Select "Use custom image" and choose your `.wic.bz2` file
3. Select your SD card and write

## Customization

### Adding Your Own Layer
1. Create your custom layer:
```bash
bitbake-layers create-layer ../meta/meta-mycustomlayer
```

2. Add recipes and configurations to your layer

3. Add the layer to your `conf/bblayers.conf`:
```bash
bitbake-layers add-layer ../meta/meta-mycustomlayer
```

### Available Images
This manifest supports building various image types:
- `core-image-minimal`: Minimal bootable image
- `core-image-base`: Base image with package management
- `core-image-full-cmdline`: Full-featured command-line image
- `core-image-x11`: Image with X11 support
- `core-image-weston`: Image with Wayland/Weston compositor

### Build Options
Configure additional features in `conf/local.conf`:
```bash
# Enable systemd instead of SysV init
DISTRO_FEATURES:append = " systemd"
VIRTUAL-RUNTIME_init_manager = "systemd"

# Add WiFi support
IMAGE_INSTALL:append = " linux-firmware-rpidistro-bcm43430 linux-firmware-rpidistro-bcm43455"

# Add SSH server
IMAGE_INSTALL:append = " openssh"

# Add development tools
IMAGE_INSTALL:append = " git python3 python3-pip"
```

## Troubleshooting

### Common Issues

1. **Build fails with "No space left on device"**
   - Ensure you have sufficient disk space (100GB+ recommended)
   - Consider setting `TMPDIR` to a location with more space

2. **Network timeouts during repo sync or bitbake**
   - Check your internet connection
   - Retry the command, Yocto will resume from where it left off

3. **Permission denied errors**
   - Ensure your user has proper permissions
   - Don't run bitbake as root

4. **Missing dependencies**
   - Install all required packages listed in Prerequisites
   - Update your package manager cache

### Getting Help
- [Yocto Project Documentation](https://docs.yoctoproject.org/)
- [Yocto Project Mailing Lists](https://lists.yoctoproject.org/)
- [meta-raspberrypi Documentation](https://meta-raspberrypi.readthedocs.io/)

## Contributing

1. Fork this repository
2. Create a feature branch
3. Make your changes
4. Test the build
5. Submit a pull request

## License

This manifest repository follows the same licensing as the Yocto Project. Individual layers may have their own licenses. Please refer to each layer's documentation for specific licensing information.

## Manifest Structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <!-- Remotes define where to fetch repositories from -->
  <remote name="yocto" fetch="https://git.yoctoproject.org/" />
  <remote name="github" fetch="https://github.com/" />

  <!-- Default settings for all projects -->
  <default remote="yocto" revision="scarthgap" sync-j="4" />

  <!-- Core Yocto Project -->
  <project name="poky" path="poky" />

  <!-- Additional layers -->
  <project name="meta-openembedded" path="meta/meta-openembedded" />
  <project name="meta-security" path="meta/meta-security" />
  <project name="meta-virtualization" path="meta/meta-virtualization" />
  <project name="agherzan/meta-raspberrypi" remote="github" path="meta/meta-raspberrypi" revision="scarthgap" />
  
  <!-- Personal development tools -->
  <project name="vsnam19/yocto-docker-env" remote="github" path="docker-env" revision="main" />
</manifest>
```

The manifest file is named `default.xml` to follow repo tool conventions.

### Remote Structure
The manifest uses two remotes for simplicity:

- **`yocto`**: Official Yocto Project repositories (git.yoctoproject.org)
  - Core Poky distribution and official meta layers
- **`github`**: All GitHub-hosted repositories (github.com)
  - Community projects like meta-raspberrypi
  - Personal repositories like yocto-docker-env

## Version Information

- **Yocto Release**: Scarthgap (5.0)
- **Manifest Version**: 1.0
- **Last Updated**: September 2025

---

**Note**: This is a development manifest. For production use, consider pinning specific commit SHAs instead of using branch names to ensure reproducible builds.