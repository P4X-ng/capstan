# Image Management: Pulling, Building, and Reusing OSv Images

This comprehensive guide covers everything you need to know about managing OSv images with Capstan, including where images come from, how to pull them effectively, and how to build on top of existing images.

## Table of Contents
- [Understanding OSv Images](#understanding-osv-images)
- [Image Sources and Repositories](#image-sources-and-repositories)
- [Pulling Images](#pulling-images)
- [Repository Schemas](#repository-schemas)
- [Building Custom Images](#building-custom-images)
- [Reusing and Layering Images](#reusing-and-layering-images)
- [Image Management Commands](#image-management-commands)
- [Best Practices](#best-practices)

## Understanding OSv Images

### What are OSv Images?
OSv images are complete, bootable unikernel instances that contain:
- **OSv Kernel**: The minimal operating system
- **Runtime Environment**: Language runtimes (Python, Java, Node.js, etc.)
- **Application Code**: Your actual application files
- **Dependencies**: Libraries and packages required by your application
- **Configuration**: Boot parameters and environment settings

### Image Types

#### 1. Base Images
Minimal OSv kernels that serve as building blocks:
```bash
# Core base images
osv-loader          # Minimal OSv kernel only
osv-base           # OSv with basic utilities
cloudius/osv-base  # Community-maintained base
```

#### 2. Runtime Images
Base images with specific language runtimes pre-installed:
```bash
# Runtime-specific bases
cloudius/osv-openjdk8    # OSv + OpenJDK 8
cloudius/osv-python3     # OSv + Python 3.x
cloudius/osv-node        # OSv + Node.js
```

#### 3. Application Images
Complete images with applications ready to run:
```bash
# Your custom application images
my-python-app      # Your Python application
my-java-service    # Your Java service
```

## Image Sources and Repositories

### 1. GitHub Repository (Default Source)

**URL**: `https://github.com/cloudius-systems/osv/releases`

**What it contains**:
- Official OSv releases
- Core base images
- Essential runtime packages
- Verified and tested components

**Usage**:
```bash
# Default behavior - pulls from GitHub
capstan pull cloudius/osv-base

# Explicitly specify GitHub (same as default)
capstan --release-tag latest pull cloudius/osv-base

# Pull specific version
capstan --release-tag v0.54.0 pull cloudius/osv-base
```

**Release Tag Options**:
- `latest`: Most recent stable release
- `v0.54.0`: Specific version number
- `any`: First available version found

### 2. S3 Repository (MIKELANGELO Project)

**URL**: `https://mikelangelo-capstan.s3.amazonaws.com`

**What it contains**:
- Extended package collection
- Experimental packages
- Additional runtime environments
- Community contributions

**Usage**:
```bash
# Pull from S3 repository
capstan --s3 pull osv.python3x

# Set S3 as default in config
echo "repo_url: https://mikelangelo-capstan.s3.amazonaws.com" > ~/.capstan/config.yaml
```

### 3. Custom Repositories

You can host your own repository with the same structure:

**Configuration**:
```bash
# Via command line
capstan -u https://my-repo.example.com pull my-image

# Via environment variable
export CAPSTAN_REPO_URL=https://my-repo.example.com
capstan pull my-image

# Via config file
echo "repo_url: https://my-repo.example.com" > ~/.capstan/config.yaml
```

## Pulling Images

### Basic Pulling Commands

```bash
# Pull a base image
capstan pull cloudius/osv-base

# Pull with specific repository
capstan --s3 pull osv.python3x

# Pull specific version
capstan --release-tag v0.54.0 pull cloudius/osv-base

# Pull and show verbose output
capstan pull -v cloudius/osv-base
```

### Understanding Pull Process

When you run `capstan pull`, here's what happens:

1. **Repository Selection**: Capstan determines which repository to use
2. **Image Lookup**: Searches for the image in the repository
3. **Metadata Download**: Downloads image metadata and manifest
4. **Image Download**: Downloads the actual image file
5. **Local Storage**: Stores image in `~/.capstan/repository/`
6. **Index Update**: Updates local image index

### Pull Strategies

#### Strategy 1: Pull Base Images First
```bash
# Pull base images you'll build upon
capstan pull cloudius/osv-base
capstan pull cloudius/osv-openjdk8
capstan --s3 pull osv.python3x
```

#### Strategy 2: Pull on Demand
```bash
# Use --pull-missing flag during compose
capstan package compose my-app --pull-missing
```

#### Strategy 3: Bulk Pull for Offline Development
```bash
# Pull all commonly used images
capstan pull cloudius/osv-base
capstan --s3 pull osv.python3x
capstan --s3 pull osv.java
capstan --s3 pull osv.cli
capstan --s3 pull osv.httpserver
```

## Repository Schemas

### GitHub Repository Schema

The GitHub repository follows OSv's release structure:

```
https://github.com/cloudius-systems/osv/releases/
├── v0.54.0/
│   ├── osv-loader.qemu.gz          # Base kernel
│   ├── osv.bootstrap.mpm           # Bootstrap package
│   ├── osv.python3x.mpm           # Python runtime
│   └── ...
├── v0.53.0/
└── latest -> v0.54.0
```

**Effective Pulling Schema**:
```bash
# Pattern: capstan [--release-tag TAG] pull IMAGE_NAME
capstan --release-tag v0.54.0 pull osv.python3x

# Capstan constructs URL:
# https://api.github.com/repos/cloudius-systems/osv/releases
# Then downloads from:
# https://github.com/cloudius-systems/osv/releases/download/v0.54.0/osv.python3x.mpm
```

### S3 Repository Schema

The S3 repository has a specific directory structure:

```
https://mikelangelo-capstan.s3.amazonaws.com/
├── mike/
│   ├── osv-loader/
│   │   ├── index.yaml              # Base image metadata
│   │   └── osv-loader.qemu.gz      # Base kernel image
│   └── osv-zfs-builder/
│       ├── index.yaml
│       └── osv-zfs-builder.qemu.gz
└── packages/
    ├── osv.bootstrap.mpm           # Package file (tar.gz)
    ├── osv.bootstrap.yaml          # Package metadata
    ├── osv.python3x.mpm
    ├── osv.python3x.yaml
    └── ...
```

**Effective Pulling Schema for S3**:
```bash
# For base images:
# URL: {S3_URL}/mike/{IMAGE_NAME}/index.yaml
# File: {S3_URL}/mike/{IMAGE_NAME}/{IMAGE_NAME}.qemu.gz

# For packages:
# Metadata: {S3_URL}/packages/{PACKAGE_NAME}.yaml
# Package: {S3_URL}/packages/{PACKAGE_NAME}.mpm
```

### Package Metadata Schema

Every package has a YAML metadata file:

```yaml
# Example: osv.python3x.yaml
name: osv.python3x
title: Python 3.x Runtime for OSv
author: OSv Team
version: 3.8.10
created: 2023-01-15T10:30:00Z
description: |
  Complete Python 3.x runtime environment including:
  - Python 3.8.10 interpreter
  - Standard library modules
  - pip package manager
  - Essential development tools
platform: x86_64
require:
  - osv.bootstrap
provides:
  - python3
  - pip3
  - python3-config
```

### Base Image Index Schema

Base images have an index.yaml file:

```yaml
# Example: mike/osv-loader/index.yaml
format_version: 1
version: 0.54.0
created: 2023-01-15T10:30:00Z
description: OSv base loader image
platform: x86_64
file: osv-loader.qemu.gz
size: 52428800
checksum:
  md5: d41d8cd98f00b204e9800998ecf8427e
  sha256: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
```

## Building Custom Images

### Method 1: Using Capstanfile (Docker-like)

Create a `Capstanfile`:
```yaml
# Capstanfile
base: cloudius/osv-base

# Optional build command
build: |
  pip3 install -r requirements.txt --target /app/lib
  echo "Build completed"

# Boot command
cmdline: python3 /app/main.py

# File mappings
files:
  /app/main.py: src/main.py
  /app/config.json: config/production.json
  /app/requirements.txt: requirements.txt

# Optional: specify rootfs directory
rootfs: ./rootfs
```

Build the image:
```bash
capstan build my-custom-app
```

### Method 2: Using Package System

1. **Initialize package structure**:
```bash
capstan package init --name my-app --title "My Application"
capstan runtime init --runtime native
```

2. **Configure dependencies** in `meta/package.yaml`:
```yaml
name: my-app
title: My Application
author: Your Name
require:
  - osv.python3x
  - osv.httpserver
```

3. **Configure runtime** in `meta/run.yaml`:
```yaml
runtime: native
config_set:
  default:
    bootcmd: python3 /app/main.py
    env:
      PYTHONPATH: /app
```

4. **Compose the image**:
```bash
capstan package compose my-app --pull-missing
```

## Reusing and Layering Images

### Understanding Image Layering

OSv images can be built in layers, similar to Docker:

```
Your Application Image
├── Application Files (your code)
├── Runtime Layer (Python, Java, etc.)
├── Utility Layer (CLI tools, libraries)
└── Base Layer (OSv kernel)
```

### Strategy 1: Building on Runtime Images

Start with a runtime-specific base:
```yaml
# Capstanfile
base: cloudius/osv-python3  # Already has Python installed

cmdline: python3 /app/main.py

files:
  /app/main.py: main.py
  /app/requirements.txt: requirements.txt
```

### Strategy 2: Creating Intermediate Images

Build reusable intermediate images:

1. **Create base application image**:
```yaml
# base-python-app/Capstanfile
base: cloudius/osv-base

build: |
  pip3 install flask gunicorn requests --target /app/lib

files:
  /app/lib: ./empty  # Placeholder, populated by build command
```

2. **Build and save**:
```bash
cd base-python-app
capstan build my-python-base
```

3. **Use as base for specific applications**:
```yaml
# specific-app/Capstanfile
base: my-python-base

cmdline: python3 /app/main.py

files:
  /app/main.py: main.py
  /app/config.json: config.json
```

### Strategy 3: Package-based Layering

Create reusable packages:

1. **Create shared library package**:
```bash
mkdir shared-libs
cd shared-libs
capstan package init --name shared-libs --title "Shared Libraries"
# Add common libraries and dependencies
capstan package build
capstan package import
```

2. **Use in multiple applications**:
```yaml
# meta/package.yaml for app1
name: app1
require:
  - osv.python3x
  - shared-libs

# meta/package.yaml for app2  
name: app2
require:
  - osv.python3x
  - shared-libs
```

### Image Inheritance Patterns

#### Pattern 1: Linear Inheritance
```
osv-base → python-base → flask-base → my-web-app
```

#### Pattern 2: Composition
```
osv-base + python3x + httpserver + cli → my-full-app
```

#### Pattern 3: Specialized Bases
```
osv-base → data-science-base (numpy, pandas, scipy)
         → ml-base (tensorflow, pytorch)
         → my-ml-app
```

## Image Management Commands

### Listing Images
```bash
# List local images
capstan images

# List with detailed information
capstan images -v

# Search remote repositories
capstan search
capstan --s3 search python
```

### Image Information
```bash
# Show image details
capstan info my-app

# Show image size and layers
capstan images my-app
```

### Cleaning Up Images
```bash
# Remove specific image
capstan rmi my-old-app

# Remove unused images (be careful!)
capstan images --filter dangling=true -q | xargs capstan rmi
```

### Importing and Exporting
```bash
# Import image from file
capstan import my-app.tar my-app

# Export image (if supported)
# Note: Direct export may not be available, use package build instead
```

## Best Practices

### 1. Image Naming Conventions
```bash
# Use descriptive, hierarchical names
my-company/python-base
my-company/web-app
my-company/data-processor

# Include version tags when appropriate
my-app:v1.0.0
my-app:latest
```

### 2. Layer Optimization
- **Minimize layers**: Combine related operations in single build steps
- **Order layers by change frequency**: Put stable dependencies first
- **Use .capstanignore**: Exclude unnecessary files

### 3. Dependency Management
```yaml
# Pin specific versions for reproducibility
require:
  - osv.python3x:3.8.10
  - osv.httpserver:0.54.0
```

### 4. Build Optimization
```yaml
# Optimize build commands
build: |
  # Install dependencies in single layer
  pip3 install -r requirements.txt --target /app/lib --no-cache-dir
  # Clean up build artifacts
  rm -rf /tmp/* /var/tmp/*
```

### 5. Security Considerations
- **Use official base images** when possible
- **Verify image checksums** for critical applications
- **Keep images updated** with latest security patches
- **Minimize attack surface** by using minimal base images

### 6. Development Workflow
```bash
# Development cycle
1. capstan pull base-image          # Get latest base
2. # Develop application
3. capstan build my-app             # Build image
4. capstan run my-app               # Test locally
5. # Iterate as needed
6. capstan package build            # Create distributable package
```

This comprehensive guide provides everything you need to effectively manage OSv images with Capstan. The key is to understand the layering concept and choose the right strategy for your specific use case.