# Image and Package Management Guide

This guide covers everything you need to know about managing images and packages in Capstan, including where they come from, how to pull them, and how to create your own.

## Table of Contents

- [Understanding the Repository System](#understanding-the-repository-system)
- [Working with Images](#working-with-images)
- [Working with Packages](#working-with-packages)
- [Creating Custom Base Images](#creating-custom-base-images)
- [Creating Your Own Packages](#creating-your-own-packages)
- [Repository Configuration](#repository-configuration)

## Understanding the Repository System

Capstan uses a two-tier system for storing and retrieving components:

### Remote Repositories

**1. OSv GitHub Releases Repository (Default, Recommended)**

This is the primary source for OSv images and packages:

- **URL**: https://github.com/cloudius-systems/osv/releases
- **Access**: No authentication required
- **Content**: Official OSv kernels and standard packages
- **Updates**: Actively maintained with regular releases

To use GitHub releases:
```bash
# Pull from any available release (default)
capstan pull cloudius/osv-base

# Pull from latest release
capstan --release-tag latest pull cloudius/osv-base

# Pull from specific release
capstan --release-tag v0.54.0 pull cloudius/osv-base
```

**2. S3 Repository (Legacy)**

The original MIKELANGELO project repository:

- **URL**: https://mikelangelo-capstan.s3.amazonaws.com/
- **Access**: No authentication required
- **Content**: Legacy packages and images
- **Status**: No longer actively maintained

To use S3 repository:
```bash
capstan --s3 pull cloudius/osv-base
capstan --s3 package pull osv.bootstrap
```

### Local Repository

Your local machine stores downloaded and created images/packages:

**Location**: `~/.capstan/`

```
~/.capstan/
├── repository/          # Local images
│   ├── cloudius/
│   │   └── osv-base/
│   ├── my-app/
│   └── ...
└── packages/            # Local packages
    ├── osv.bootstrap/
    ├── osv.python3x/
    └── ...
```

## Working with Images

### Listing Images

**List local images:**
```bash
capstan images
# or
capstan i
```

Example output:
```
Name                      Description              Version    Created
cloudius/osv-base         OSv Base Image           0.54.0     2019-09-16
cloudius/osv-openjdk8     OSv with OpenJDK 8       0.54.0     2019-09-16
my-python-app             My Python Application    -          2024-01-15
```

### Pulling Images

**Pull base images:**

```bash
# Pull OSv base image from GitHub releases
capstan pull cloudius/osv-base

# Pull from specific release
capstan --release-tag v0.54.0 pull cloudius/osv-base

# Pull from latest release
capstan --release-tag latest pull cloudius/osv-base
```

**Understanding Image Names:**

Images follow the format: `namespace/image-name`

Common base images:
- `cloudius/osv-base` - Minimal OSv kernel
- `cloudius/osv-openjdk8` - OSv with Java runtime
- `cloudius/osv-openjdk10` - OSv with Java 10
- `mike/osv-loader` - Legacy base loader (S3 only)

### Image Information

**Get detailed information about an image:**

```bash
capstan info my-app
```

Output shows:
- Image format (QCOW2)
- Virtual size
- Actual size on disk
- Boot command line

### Searching for Images

While Capstan doesn't have a direct "image search" command, you can search for packages that include complete images:

```bash
capstan --release-tag latest package search
```

### Removing Images

**Delete an image:**

```bash
capstan rmi my-app
```

**Delete multiple images:**

```bash
capstan rmi app1 app2 app3
```

### Importing and Exporting Images

**Export an image:**

```bash
# Export to QCOW2 file
capstan export my-app -o my-app.qcow2

# Export and compress
capstan export my-app -o my-app.qcow2.gz
```

**Import an image:**

```bash
capstan import my-app -f my-app.qcow2

# Import with different name
capstan import new-name -f my-app.qcow2
```

This is useful for:
- Sharing images with team members
- Backing up images
- Deploying to different environments
- Moving between development machines

## Working with Packages

### Understanding Packages

Packages are compressed archives (`.mpm` files) containing:
- Application binaries and libraries
- Configuration files
- Metadata (`.yaml` descriptor)
- Dependencies on other packages

### Searching for Packages

**Search GitHub releases:**

```bash
# Search all packages
capstan --release-tag latest package search

# Search for specific package
capstan --release-tag latest package search python

# Search in specific release version
capstan --release-tag v0.54.0 package search
```

Example output:
```
Release   Name                    Description                      Version    Created
v0.54.0   osv.bootstrap          OSv Bootstrap                    0.54.0     2019-09-16
v0.54.0   osv.cli                OSv Command Line                 0.54.0     2019-09-16
v0.54.0   osv.python3x           Python 3.x Runtime               0.54.0     2019-09-16
v0.54.0   osv.run-java           Run Java wrapper                 0.54.0     2019-09-16
v0.54.0   osv.run-go             Run Golang wrapper               0.54.0     2019-09-16
```

**Search S3 repository:**

```bash
capstan --s3 package search
```

### Listing Local Packages

```bash
capstan package list
```

Example output:
```
Name                  Description                    Version    Created
osv.bootstrap        OSv Bootstrap                  0.54.0     2019-09-16
osv.cli              OSv Command Line Interface     0.54.0     2019-09-16
osv.python3x         Python 3.x Runtime             0.54.0     2019-09-16
osv.httpserver-api   OSv httpserver with APIs       0.54.0     2019-09-16
```

### Pulling Packages

**Pull a specific package:**

```bash
# From GitHub releases
capstan --release-tag latest package pull osv.python3x

# From S3 repository
capstan --s3 package pull python-2.7
```

**Pull with dependencies:**

When you require a package in your `meta/package.yaml`, Capstan can automatically pull missing packages:

```yaml
# meta/package.yaml
name: my-python-app
require:
  - osv.python3x
  - osv.httpserver-api
```

Then compose with auto-pull:
```bash
capstan package compose my-app --pull-missing
# or short form
capstan package compose my-app -p
```

### Package Information

**Describe a package:**

```bash
capstan package describe osv.python3x
```

Shows:
- Package name and description
- Version and creation date
- Dependencies (required packages)
- Platform information

### Updating Packages

**Update all local packages from remote:**

```bash
capstan package update
```

This checks the remote repository and downloads newer versions of packages you have locally.

## Creating Custom Base Images

You can create your own base images to include specific OSv features or configurations.

### Using OSv Build Script (Recommended)

The easiest way is using OSv's `build-capstan-base-image` script:

```bash
# Clone OSv repository
git clone https://github.com/cloudius-systems/osv.git
cd osv

# Build base image with specific apps
./scripts/build-capstan-base-image \
  cloudius/python3 \
  python3x \
  'OSv base with Python 3'
```

This creates a base image and imports it to your local Capstan repository.

### Custom Base Image with Specific Features

**Example: Base image with Python 3 and httpserver**

```bash
cd osv

# Build with multiple OSv apps
./scripts/build-capstan-base-image \
  "cloudius/python3 cloudius/httpserver" \
  my-python-base \
  'Custom Python 3 base with httpserver'
```

**Available OSv apps** (in `osv/apps/` directory):
- `cloudius/python3` - Python 3.x runtime
- `cloudius/openjdk8-zulu-compact1` - Java 8 runtime
- `cloudius/node` - Node.js runtime
- `cloudius/httpserver` - HTTP REST server
- `cloudius/cli` - Command-line interface

### Using Capstanfile for Base Images

You can also create base images using Capstanfile:

```yaml
# Capstanfile
base: cloudius/osv-base

# Include additional tools
files:
  /usr/bin/tool: ./tool
  /etc/config: ./config

# Optional: run a setup command
build: ./setup.sh
```

Build the base:
```bash
capstan build my-custom-base
```

## Creating Your Own Packages

### Package Structure

A Capstan package has this structure:

```
my-package/
├── meta/
│   ├── package.yaml    # Package metadata
│   └── run.yaml        # Runtime configuration (optional)
├── bin/                # Binaries
├── lib/                # Libraries
└── etc/                # Configuration files
```

### Creating a Package

**Step 1: Initialize package structure**

```bash
mkdir my-custom-package
cd my-custom-package

capstan package init \
  --name "com.example.mypackage" \
  --title "My Custom Package" \
  --author "Your Name" \
  --require osv.bootstrap
```

**Step 2: Add your files**

```bash
# Add binaries, libraries, config files
mkdir -p bin lib etc
cp /path/to/mybinary bin/
cp /path/to/mylib.so lib/
cp /path/to/config.conf etc/
```

**Step 3: Create run.yaml (if needed)**

```bash
capstan runtime init --runtime native
```

Edit `meta/run.yaml`:
```yaml
runtime: native

config_set:
  default:
    bootcmd: "/bin/mybinary"
```

**Step 4: Build the package**

```bash
capstan package build
```

This creates `my-custom-package.mpm` in the current directory.

**Step 5: Import to local repository**

```bash
capstan package import my-custom-package.mpm
```

### Package with Dependencies

Create a package that depends on other packages:

```yaml
# meta/package.yaml
name: com.example.myapp
title: My Application
author: Your Name
require:
  - osv.python3x
  - osv.httpserver-api
```

When this package is used, Capstan automatically includes the required packages.

### Sharing Packages

**Export package:**

```bash
capstan package build
# Creates: com.example.mypackage.mpm
```

**Share the .mpm file** with others who can import it:

```bash
capstan package import com.example.mypackage.mpm
```

## Repository Configuration

### Setting Default Repository

You can configure Capstan to use a different default repository:

**Using configuration file:**

Create `~/.capstan/config.yaml`:

```yaml
# Use specific S3 URL
default_repo: https://my-custom-repo.s3.amazonaws.com/

# Or use GitHub releases with specific tag
release_tag: v0.54.0
```

**Using environment variables:**

```bash
export CAPSTAN_REPO_URL=https://my-custom-repo.s3.amazonaws.com/
export CAPSTAN_RELEASE_TAG=latest
```

**Using command-line flags:**

```bash
# Custom S3 URL
capstan -u https://my-custom-repo.s3.amazonaws.com/ pull cloudius/osv-base

# Specific release tag
capstan --release-tag v0.54.0 pull cloudius/osv-base
```

### Hosting Your Own Repository

To host your own Capstan repository, you need:

**For GitHub Releases:**
1. Fork OSv repository
2. Build images and packages
3. Create releases with assets
4. Point Capstan to your repository

**For S3-style Repository:**

Create this structure:

```
my-repo/
├── mike/
│   ├── osv-loader/
│   │   ├── index.yaml
│   │   └── osv-loader.qemu.gz
└── packages/
    ├── mypackage.mpm
    └── mypackage.yaml
```

**index.yaml format:**
```yaml
format_version: "1"
version: "0.54.0"
created: "2019-09-16"
description: "OSv Bootloader"
platform: "Ubuntu 19.04"
```

**package.yaml format:**
```yaml
name: mypackage
title: "My Package"
author: "Your Name"
version: "1.0.0"
require:
  - osv.bootstrap
platform: "Ubuntu 19.04"
created: "2024-01-15"
```

Host on S3, HTTP server, or any file server, then use:
```bash
capstan -u https://my-repo.example.com/ pull cloudius/osv-base
```

## Best Practices

### Image Management

1. **Use descriptive names**: `mycompany/app-name` instead of `app`
2. **Tag versions**: Keep multiple versions if needed
3. **Clean up**: Remove unused images with `capstan rmi`
4. **Backup important images**: Export before major changes

### Package Management

1. **Pin dependencies**: Specify exact package versions when needed
2. **Use latest release tag**: `--release-tag latest` for most recent packages
3. **Update regularly**: Run `capstan package update` periodically
4. **Document requirements**: List required packages in documentation

### Repository Usage

1. **Prefer GitHub releases**: More up-to-date than S3
2. **Cache locally**: Pull packages once, use `--pull-missing` sparingly
3. **Check connectivity**: Ensure internet access for remote pulls
4. **Use specific releases**: For reproducible builds, pin release tag

## Troubleshooting

**Cannot pull image/package:**
- Check internet connection
- Try: `capstan --release-tag latest pull cloudius/osv-base`
- Verify repository URL: `capstan config`

**Package not found:**
- Search available packages: `capstan --release-tag latest package search`
- Check package name spelling
- Try S3 repository: `capstan --s3 package search`

**Disk space issues:**
- List images: `capstan images`
- Remove unused: `capstan rmi <image-name>`
- Clean package cache: `rm -rf ~/.capstan/packages/*` (careful!)

**Import/export fails:**
- Check file permissions
- Verify QCOW2 format: `file image.qcow2`
- Check available disk space

## Reference

### Common Image Commands

```bash
capstan images                    # List local images
capstan pull <image>              # Pull remote image
capstan rmi <image>               # Remove image
capstan info <image>              # Show image info
capstan import <name> -f <file>   # Import image
capstan export <name> -o <file>   # Export image
```

### Common Package Commands

```bash
capstan package list                    # List local packages
capstan package search [name]           # Search remote
capstan package pull <name>             # Pull package
capstan package describe <name>         # Show package info
capstan package update                  # Update all packages
capstan package build                   # Build package
capstan package import <file>           # Import package
```

### Global Flags

```bash
--release-tag latest    # Use latest GitHub release
--release-tag v0.54.0   # Use specific GitHub release
--s3                    # Use S3 repository
-u <url>                # Use custom repository URL
```
