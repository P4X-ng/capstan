# Schema Reference: Complete Configuration Guide

This document provides a comprehensive reference for all Capstan configuration files, schemas, and their options. Use this as a definitive guide for creating effective Capstan configurations.

## Table of Contents
- [Configuration File Overview](#configuration-file-overview)
- [Capstanfile Schema](#capstanfile-schema)
- [Package Configuration (meta/package.yaml)](#package-configuration-metapackageyaml)
- [Runtime Configuration (meta/run.yaml)](#runtime-configuration-metarunyaml)
- [Capstan Global Configuration](#capstan-global-configuration)
- [Repository Schemas](#repository-schemas)
- [Environment Variables](#environment-variables)
- [Best Practices](#best-practices)

## Configuration File Overview

Capstan uses several configuration files to define how applications are built and run:

| File | Purpose | Required | Location |
|------|---------|----------|----------|
| `Capstanfile` | Docker-like image building | Optional | Project root |
| `meta/package.yaml` | Package metadata and dependencies | Required for packages | `meta/` directory |
| `meta/run.yaml` | Runtime configuration | Required for packages | `meta/` directory |
| `~/.capstan/config.yaml` | Global Capstan settings | Optional | User home |
| `.capstanignore` | Files to exclude from packaging | Optional | Project root |

## Capstanfile Schema

The Capstanfile uses a Docker-like syntax for building OSv images.

### Complete Schema
```yaml
# Base image to build upon (required)
base: string

# Boot command line (required)
cmdline: string

# Optional build command
build: string | array

# File mappings (optional)
files:
  "/target/path": "source/path"
  "/another/target": "source/file"

# Root filesystem directory (optional)
rootfs: string

# RPM package installation (optional)
rpm-base:
  name: string
  version: string
  release: string
  arch: string
```

### Field Descriptions

#### `base` (required)
Specifies the base OSv image to build upon.

**Type**: `string`
**Examples**:
```yaml
base: cloudius/osv-base           # Community base image
base: osv-loader                  # Minimal OSv kernel
base: my-custom-base              # Custom base image
```

#### `cmdline` (required)
The command line that OSv will execute when the image boots.

**Type**: `string`
**Examples**:
```yaml
cmdline: /usr/bin/python3 /app/main.py
cmdline: java -cp /app MyApplication
cmdline: /tools/hello.so
```

#### `build` (optional)
Commands to execute during the image build process.

**Type**: `string` or `array`
**Examples**:
```yaml
# Single command
build: make

# Multiple commands as string
build: |
  pip3 install -r requirements.txt --target /app/lib
  chmod +x /app/scripts/*.sh

# Multiple commands as array
build:
  - pip3 install -r requirements.txt --target /app/lib
  - chmod +x /app/scripts/*.sh
  - rm -rf /tmp/*
```

#### `files` (optional)
Maps local files to paths in the OSv image.

**Type**: `object`
**Format**: `"/target/path": "source/path"`
**Examples**:
```yaml
files:
  /app/main.py: src/main.py
  /app/config.json: config/production.json
  /usr/lib/mylib.so: build/mylib.so
  
# Using basename substitution with &
files:
  /app/app.jar: build/&  # Expands to build/app.jar
```

#### `rootfs` (optional)
Specifies a directory whose contents will be copied to the image root.

**Type**: `string`
**Default**: `ROOTFS` (if directory exists)
**Examples**:
```yaml
rootfs: ./filesystem
rootfs: /path/to/rootfs
```

#### `rpm-base` (optional)
Installs RPM packages during build.

**Type**: `object`
**Fields**:
- `name`: Package name
- `version`: Package version
- `release`: Package release
- `arch`: Architecture

**Example**:
```yaml
rpm-base:
  name: java-1.8.0-openjdk
  version: 1.8.0.232
  release: b09.0.el7_7
  arch: x86_64
```

### Complete Capstanfile Examples

#### Python Web Application
```yaml
base: cloudius/osv-base

build: |
  pip3 install -r requirements.txt --target /app/lib --no-cache-dir
  find /app -name "*.pyc" -delete

cmdline: python3 /app/main.py

files:
  /app/main.py: src/main.py
  /app/requirements.txt: requirements.txt
  /app/config/: config/

rootfs: ./static_files
```

#### Java Application
```yaml
base: cloudius/osv-base

build: |
  javac -cp /app/lib/*:/app src/*.java -d /app
  jar cf /app/myapp.jar -C /app .

cmdline: java -cp /app/myapp.jar:/app/lib/* com.example.Main

files:
  /app/src/: src/
  /app/lib/: lib/
```

## Package Configuration (meta/package.yaml)

Defines package metadata, dependencies, and basic information.

### Complete Schema
```yaml
# Package identification (required)
name: string
title: string
author: string

# Version information (optional)
version: string
created: string (ISO 8601 date)

# Dependencies (optional)
require:
  - package-name
  - package-name:version

# Provided capabilities (optional)
provides:
  - capability-name

# Platform information (optional)
platform: string

# Description (optional)
description: string
```

### Field Descriptions

#### `name` (required)
Unique package identifier using reverse domain notation.

**Type**: `string`
**Format**: Reverse domain notation recommended
**Examples**:
```yaml
name: com.example.myapp
name: my-python-web-server
name: data-processor
```

#### `title` (required)
Human-readable package title.

**Type**: `string`
**Examples**:
```yaml
title: My Python Web Server
title: Data Processing Service
title: Admin Tools
```

#### `author` (required)
Package author information.

**Type**: `string`
**Examples**:
```yaml
author: John Doe
author: John Doe (john@example.com)
author: Example Corp <contact@example.com>
```

#### `version` (optional)
Package version string.

**Type**: `string`
**Examples**:
```yaml
version: 1.0.0
version: 2.1.3-beta
version: latest
```

#### `require` (optional)
List of required packages and their versions.

**Type**: `array`
**Format**: `package-name` or `package-name:version`
**Examples**:
```yaml
require:
  - osv.python3x
  - osv.httpserver
  - osv.cli:0.54.0
  - my-shared-lib:latest
```

#### `provides` (optional)
List of capabilities this package provides.

**Type**: `array`
**Examples**:
```yaml
provides:
  - python3
  - web-server
  - data-processing
```

### Complete Package Examples

#### Python Web Application Package
```yaml
name: com.example.python-web-app
title: Python Web Application
author: Development Team <dev@example.com>
version: 1.2.0
created: 2023-12-01T10:30:00Z
platform: x86_64

description: |
  A comprehensive Python web application built with Flask,
  including API endpoints, database connectivity, and
  background task processing.

require:
  - osv.python3x
  - osv.httpserver
  - osv.bootstrap

provides:
  - web-api
  - task-processor
```

#### Java Microservice Package
```yaml
name: com.example.java-microservice
title: Java Microservice
author: Backend Team
version: 2.0.1
platform: x86_64

require:
  - osv.java
  - osv.httpserver

provides:
  - rest-api
  - data-service
```

## Runtime Configuration (meta/run.yaml)

Defines how the package should be executed, including runtime selection and boot configurations.

### Complete Schema
```yaml
# Runtime selection (required)
runtime: string

# Simple configuration (option 1)
# Runtime-specific fields...

# Named configurations (option 2)
config_set:
  config-name:
    # Runtime-specific fields...
  another-config:
    # Runtime-specific fields...

# Default configuration (optional, for named configs)
config_set_default: string
```

### Runtime Types

#### Native Runtime
The most flexible runtime that allows arbitrary commands.

```yaml
runtime: native

# Simple configuration
bootcmd: string
env:
  VAR_NAME: value

# Named configuration
config_set:
  config-name:
    bootcmd: string
    env:
      VAR_NAME: value
```

**Fields**:
- `bootcmd`: Command to execute (required)
- `env`: Environment variables (optional)

**Examples**:
```yaml
runtime: native

config_set:
  web-server:
    bootcmd: python3 /app/server.py
    env:
      PYTHONPATH: /app:/app/lib
      PORT: "8080"
      DEBUG: "false"
      
  worker:
    bootcmd: python3 /app/worker.py
    env:
      PYTHONPATH: /app:/app/lib
      WORKER_TYPE: background
      
  admin:
    bootcmd: python3 /app/admin.py status
    env:
      PYTHONPATH: /app:/app/lib

config_set_default: web-server
```

#### Java Runtime (Legacy)
Specialized runtime for Java applications.

```yaml
runtime: java

# Simple configuration
main: string
classpath:
  - path
args:
  - argument
vmargs:
  - jvm-argument

# Named configuration
config_set:
  config-name:
    main: string
    classpath:
      - path
    args:
      - argument
    vmargs:
      - jvm-argument
```

**Fields**:
- `main`: Main class name (required)
- `classpath`: Java classpath entries (optional)
- `args`: Application arguments (optional)
- `vmargs`: JVM arguments (optional)

**Example**:
```yaml
runtime: java

config_set:
  web-server:
    main: com.example.WebServer
    classpath:
      - /app/lib
      - /app/myapp.jar
    args:
      - --port=8080
      - --config=/app/config.properties
    vmargs:
      - Xmx512m
      - Xms256m
      - Dfile.encoding=UTF-8
      
  batch-processor:
    main: com.example.BatchProcessor
    classpath:
      - /app/lib
      - /app/myapp.jar
    vmargs:
      - Xmx1g

config_set_default: web-server
```

#### Python Runtime (Legacy)
Specialized runtime for Python applications.

```yaml
runtime: python

# Simple configuration
main: string
args:
  - argument
shell: boolean

# Named configuration
config_set:
  config-name:
    main: string
    args:
      - argument
    shell: boolean
```

**Fields**:
- `main`: Python script path (required unless shell=true)
- `args`: Script arguments (optional)
- `shell`: Start interactive shell (optional)

**Example**:
```yaml
runtime: python

config_set:
  application:
    main: /app/main.py
    args:
      - --verbose
      - --config=/app/config.json
      
  shell:
    shell: true

config_set_default: application
```

### Environment Variables

Common environment variables used across runtimes:

#### Python Environment Variables
```yaml
env:
  PYTHONPATH: "/app:/app/lib"           # Python module search path
  PYTHONUNBUFFERED: "1"                # Unbuffered output
  PYTHONDONTWRITEBYTECODE: "1"         # Don't create .pyc files
  PYTHONOPTIMIZE: "2"                  # Optimization level
  PYTHONHASHSEED: "0"                  # Deterministic hash seed
  FLASK_ENV: "production"              # Flask environment
  DEBUG: "false"                       # Debug mode
```

#### Java Environment Variables
```yaml
env:
  JAVA_OPTS: "-Xmx512m -Xms256m"      # JVM options
  CLASSPATH: "/app/lib:/app/myapp.jar" # Java classpath
  JAVA_HOME: "/usr/lib/jvm/java"       # Java installation path
```

#### General Environment Variables
```yaml
env:
  PORT: "8080"                         # Application port
  HOST: "0.0.0.0"                      # Bind address
  LOG_LEVEL: "INFO"                    # Logging level
  CONFIG_FILE: "/app/config.json"      # Configuration file path
  SERVICE_MODE: "production"           # Service mode
```

## Capstan Global Configuration

Global Capstan settings stored in `~/.capstan/config.yaml`.

### Complete Schema
```yaml
# Repository configuration
repo_url: string

# Virtualization settings
disable_kvm: boolean
qemu_aio_type: string

# Release management
release_tag: string

# GitHub API settings (optional)
github_api_url: string
github_token: string
```

### Field Descriptions

#### `repo_url` (optional)
Default repository URL for pulling images and packages.

**Type**: `string`
**Default**: GitHub repository
**Examples**:
```yaml
repo_url: "https://mikelangelo-capstan.s3.amazonaws.com"
repo_url: "https://my-custom-repo.example.com"
repo_url: ""  # Use default GitHub repository
```

#### `disable_kvm` (optional)
Disable KVM acceleration for virtualization.

**Type**: `boolean`
**Default**: `false`
**Examples**:
```yaml
disable_kvm: false  # Use KVM if available
disable_kvm: true   # Force disable KVM
```

#### `qemu_aio_type` (optional)
QEMU asynchronous I/O type.

**Type**: `string`
**Default**: `threads`
**Options**: `threads`, `native`, `io_uring`
**Examples**:
```yaml
qemu_aio_type: threads   # Default
qemu_aio_type: native    # Native AIO
qemu_aio_type: io_uring  # io_uring (Linux 5.1+)
```

#### `release_tag` (optional)
Default release tag for GitHub repository pulls.

**Type**: `string`
**Default**: `any`
**Options**: `any`, `latest`, `v0.54.0`, etc.
**Examples**:
```yaml
release_tag: latest   # Latest stable release
release_tag: v0.54.0  # Specific version
release_tag: any      # Any available version
```

### Complete Configuration Example
```yaml
# ~/.capstan/config.yaml
repo_url: "https://mikelangelo-capstan.s3.amazonaws.com"
disable_kvm: false
qemu_aio_type: threads
release_tag: latest

# Optional GitHub settings
github_api_url: "https://api.github.com"
```

## Repository Schemas

### GitHub Repository Schema

GitHub repository follows OSv release structure:

```
https://api.github.com/repos/cloudius-systems/osv/releases
├── v0.54.0/
│   ├── osv-loader.qemu.gz
│   ├── osv.bootstrap.mpm
│   ├── osv.python3x.mpm
│   └── ...
├── v0.53.0/
└── latest -> v0.54.0
```

#### Release API Response Schema
```json
{
  "tag_name": "v0.54.0",
  "name": "OSv v0.54.0",
  "published_at": "2023-01-15T10:30:00Z",
  "assets": [
    {
      "name": "osv-loader.qemu.gz",
      "browser_download_url": "https://github.com/.../osv-loader.qemu.gz",
      "size": 52428800
    },
    {
      "name": "osv.python3x.mpm",
      "browser_download_url": "https://github.com/.../osv.python3x.mpm",
      "size": 104857600
    }
  ]
}
```

### S3 Repository Schema

S3 repository structure for MIKELANGELO project:

```
https://mikelangelo-capstan.s3.amazonaws.com/
├── mike/
│   ├── osv-loader/
│   │   ├── index.yaml
│   │   └── osv-loader.qemu.gz
│   └── osv-zfs-builder/
│       ├── index.yaml
│       └── osv-zfs-builder.qemu.gz
└── packages/
    ├── osv.bootstrap.mpm
    ├── osv.bootstrap.yaml
    ├── osv.python3x.mpm
    ├── osv.python3x.yaml
    └── ...
```

#### Base Image Index Schema (mike/*/index.yaml)
```yaml
format_version: 1
version: string
created: string (ISO 8601)
description: string
platform: string
file: string
size: integer
checksum:
  md5: string
  sha256: string
```

**Example**:
```yaml
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

#### Package Metadata Schema (packages/*.yaml)
```yaml
name: string
title: string
author: string
version: string
created: string (ISO 8601)
description: string
platform: string
require:
  - string
provides:
  - string
size: integer
checksum:
  md5: string
  sha256: string
```

**Example**:
```yaml
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
platform: x86_64
require:
  - osv.bootstrap
provides:
  - python3
  - pip3
size: 104857600
checksum:
  md5: 5d41402abc4b2a76b9719d911017c592
  sha256: aec070645fe53ee3b3763059376134f058cc337247c978add178b6ccdfb0019f
```

## Environment Variables

### Capstan Environment Variables

#### Repository Configuration
```bash
# Repository URL
export CAPSTAN_REPO_URL="https://my-repo.example.com"

# GitHub API settings
export CAPSTAN_GITHUB_API_URL="https://api.github.com"
export CAPSTAN_GITHUB_TOKEN="ghp_xxxxxxxxxxxx"
```

#### Virtualization Settings
```bash
# Disable KVM acceleration
export CAPSTAN_DISABLE_KVM=true

# QEMU AIO type
export CAPSTAN_QEMU_AIO_TYPE=native
```

#### Build and Runtime Settings
```bash
# Default release tag
export CAPSTAN_RELEASE_TAG=latest

# Build verbosity
export CAPSTAN_VERBOSE=true

# Default memory size
export CAPSTAN_MEMORY=2G
```

### Application Environment Variables

#### Python Applications
```bash
# Python path configuration
export PYTHONPATH="/app:/app/lib"
export PYTHONUNBUFFERED=1
export PYTHONDONTWRITEBYTECODE=1

# Flask configuration
export FLASK_ENV=production
export FLASK_APP=main.py

# Django configuration
export DJANGO_SETTINGS_MODULE=myproject.settings
```

#### Java Applications
```bash
# JVM configuration
export JAVA_OPTS="-Xmx512m -Xms256m"
export JAVA_HOME="/usr/lib/jvm/java"

# Application configuration
export CLASSPATH="/app/lib:/app/myapp.jar"
```

## Best Practices

### 1. Configuration Organization

#### Project Structure
```
my-project/
├── Capstanfile              # For Docker-like builds
├── .capstanignore          # Exclude files from packaging
├── meta/
│   ├── package.yaml        # Package metadata
│   └── run.yaml           # Runtime configuration
├── src/                   # Application source code
├── config/               # Configuration files
├── tests/               # Test files
└── docs/               # Documentation
```

#### Configuration Separation
```yaml
# Separate configurations by environment
config_set:
  development:
    bootcmd: python3 -u /app/main.py
    env:
      DEBUG: "true"
      LOG_LEVEL: "DEBUG"
      
  staging:
    bootcmd: python3 /app/main.py
    env:
      DEBUG: "false"
      LOG_LEVEL: "INFO"
      
  production:
    bootcmd: python3 -O /app/main.py
    env:
      DEBUG: "false"
      LOG_LEVEL: "WARNING"
      PYTHONOPTIMIZE: "2"
```

### 2. Version Management

#### Pin Dependencies
```yaml
# Pin specific versions for reproducibility
require:
  - osv.python3x:3.8.10
  - osv.httpserver:0.54.0
  - my-shared-lib:1.2.3
```

#### Version Your Packages
```yaml
name: com.example.myapp
version: 1.2.3
created: 2023-12-01T10:30:00Z
```

### 3. Security Considerations

#### Minimal Dependencies
```yaml
# Only include necessary packages
require:
  - osv.python3x  # Required
  # - osv.cli     # Remove if not needed
```

#### Environment Variable Security
```yaml
# Don't hardcode secrets in configuration
env:
  DATABASE_URL: "${DATABASE_URL}"  # Use environment substitution
  API_KEY: "${API_KEY}"
```

### 4. Performance Optimization

#### Memory Configuration
```yaml
config_set:
  memory-optimized:
    bootcmd: python3 -O /app/main.py
    env:
      PYTHONOPTIMIZE: "2"
      MALLOC_TRIM_THRESHOLD_: "65536"
```

#### Build Optimization
```yaml
build: |
  # Install dependencies efficiently
  pip3 install -r requirements.txt --target /app/lib --no-cache-dir
  
  # Remove unnecessary files
  find /app -name "*.pyc" -delete
  find /app -name "__pycache__" -type d -exec rm -rf {} +
  
  # Optimize permissions
  chmod -R 755 /app
```

### 5. Debugging and Development

#### Debug Configuration
```yaml
config_set:
  debug:
    bootcmd: python3 -u -v /app/main.py
    env:
      PYTHONPATH: /app:/app/lib
      PYTHONUNBUFFERED: "1"
      PYTHONVERBOSE: "1"
      DEBUG: "true"
      LOG_LEVEL: "DEBUG"
```

#### Development Helpers
```yaml
config_set:
  shell:
    bootcmd: python3 -i
    env:
      PYTHONPATH: /app:/app/lib
      
  inspect:
    bootcmd: python3 -c "import sys; print('\\n'.join(sys.path))"
    env:
      PYTHONPATH: /app:/app/lib
```

### 6. Testing Configuration

#### Test Environments
```yaml
config_set:
  test:
    bootcmd: python3 -m pytest /app/tests/ -v
    env:
      PYTHONPATH: /app:/app/lib:/app/tests
      TESTING: "true"
      
  coverage:
    bootcmd: python3 -m pytest /app/tests/ --cov=/app
    env:
      PYTHONPATH: /app:/app/lib:/app/tests
```

This comprehensive schema reference provides all the information needed to create effective Capstan configurations for any scenario. Use it as a reference when building your OSv applications and packages.