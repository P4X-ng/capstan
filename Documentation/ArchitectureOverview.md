# OSv and Capstan Architecture Overview

This document provides a comprehensive overview of the OSv unikernel ecosystem and how Capstan fits into the overall architecture to simplify application deployment.

## Table of Contents
- [What is OSv?](#what-is-osv)
- [What is Capstan?](#what-is-capstan)
- [Core Components](#core-components)
- [Architecture Diagram](#architecture-diagram)
- [Workflow Overview](#workflow-overview)
- [Component Relationships](#component-relationships)
- [Repository Ecosystem](#repository-ecosystem)

## What is OSv?

OSv is a cloud-ready operating system designed specifically for running single applications in virtualized environments. Unlike traditional operating systems, OSv is a **unikernel** - a specialized, single-address-space operating system that includes only the minimal set of features required to run your application.

### Key OSv Characteristics:
- **Single Process**: Runs only one application per instance
- **No User/Kernel Space Separation**: Applications run in kernel space for maximum performance
- **Minimal Footprint**: Only includes necessary OS components
- **Fast Boot Times**: Typically boots in seconds
- **Cloud Optimized**: Designed for virtualized environments (KVM, Xen, VMware, etc.)

### OSv Benefits:
- **Performance**: Reduced overhead and context switching
- **Security**: Smaller attack surface due to minimal components
- **Resource Efficiency**: Lower memory and CPU usage
- **Deployment Speed**: Fast boot and shutdown times

## What is Capstan?

Capstan is a command-line tool that dramatically simplifies the process of building, configuring, and running OSv unikernels. It acts as a high-level interface that abstracts away the complexity of unikernel creation.

### Capstan's Role:
- **Build Automation**: Automates the complex OSv build process
- **Package Management**: Manages precompiled components and dependencies
- **Configuration Management**: Provides simple configuration files for complex setups
- **Runtime Management**: Handles launching and managing unikernel instances
- **Repository Integration**: Connects to remote repositories for base images and packages

### Capstan Philosophy:
Instead of requiring deep unikernel knowledge, Capstan allows developers to focus on **application-oriented settings**:
- Where is the application entry point?
- What environment variables are needed?
- What dependencies does the application require?
- How should the application be configured?

## Core Components

### 1. Base Images
Base images are minimal OSv kernels that provide the foundation for your applications:
- **osv-loader**: The core OSv kernel without any applications
- **osv-base**: Basic OSv with essential utilities
- **Runtime-specific bases**: Pre-configured for specific languages (Python, Java, Node.js)

### 2. Packages (MPM - Mikelangelo Package Manager)
Packages are precompiled components that can be added to base images:
- **Runtime packages**: Language runtimes (osv.python3x, osv.java, etc.)
- **Library packages**: Common libraries and dependencies
- **Application packages**: Complete applications ready to run
- **Utility packages**: Tools and utilities (osv.cli, osv.httpserver)

### 3. Configuration Files

#### Capstanfile
Docker-like configuration for building images from scratch:
```yaml
base: cloudius/osv-base
cmdline: /usr/bin/python3 /app/main.py
build: make
files:
  /app/main.py: main.py
  /app/requirements.txt: requirements.txt
```

#### Package Configuration (meta/package.yaml)
Defines package metadata and dependencies:
```yaml
name: my-python-app
title: My Python Application
author: Developer Name
require:
  - osv.python3x
  - osv.httpserver
```

#### Runtime Configuration (meta/run.yaml)
Specifies how the application should be executed:
```yaml
runtime: native
config_set:
  default:
    bootcmd: python3 /app/main.py
  debug:
    bootcmd: python3 -u /app/main.py
    env:
      PYTHONPATH: /app
```

### 4. Repositories
Capstan connects to repositories to download base images and packages:
- **GitHub Repository**: OSv releases with official packages
- **S3 Repository**: MIKELANGELO project repository with additional packages
- **Local Repository**: Your local cache and custom packages

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Capstan Tool                             │
├─────────────────────────────────────────────────────────────────┤
│  Commands: build, compose, run, pull, package, images, etc.    │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Configuration Layer                           │
├─────────────────────────────────────────────────────────────────┤
│  • Capstanfile (Docker-like)                                   │
│  • meta/package.yaml (Package metadata)                        │
│  • meta/run.yaml (Runtime configuration)                       │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Repository Layer                             │
├─────────────────────────────────────────────────────────────────┤
│  Remote Repositories          │  Local Repository               │
│  ├─ GitHub OSv Releases       │  ├─ ~/.capstan/repository/      │
│  ├─ S3 MIKELANGELO Repo       │  ├─ ~/.capstan/packages/        │
│  └─ Custom Repositories       │  └─ ~/.capstan/instances/       │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Image Building                              │
├─────────────────────────────────────────────────────────────────┤
│  Base Image + Packages + Application Files = Complete Image    │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │ OSv Kernel  │  │  Runtime    │  │ Application │             │
│  │ (osv-loader)│  │ (Python3x)  │  │   Files     │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Runtime Execution                            │
├─────────────────────────────────────────────────────────────────┤
│  Hypervisor (KVM/QEMU, Xen, VMware, VirtualBox)               │
│  └─ OSv Unikernel Instance                                     │
│     └─ Single Application Process                              │
└─────────────────────────────────────────────────────────────────┘
```

## Workflow Overview

### 1. Development Workflow
```
Developer Code → Capstan Configuration → Image Building → Testing → Deployment
```

### 2. Image Building Process
```
Base Image → Add Packages → Add Application Files → Configure Runtime → Final Image
```

### 3. Package Composition Workflow
```
Package Definition → Dependency Resolution → File Collection → Image Assembly → Bootable Image
```

## Component Relationships

### Image Hierarchy
```
Custom Application Image
├─ Application Files (your code)
├─ Runtime Package (e.g., osv.python3x)
│  ├─ Python 3.x interpreter
│  ├─ Standard libraries
│  └─ Runtime dependencies
├─ Base Packages (e.g., osv.bootstrap)
│  ├─ Essential utilities
│  └─ Basic system components
└─ OSv Kernel (osv-loader)
   ├─ Minimal OS kernel
   ├─ Device drivers
   └─ Virtual filesystem
```

### Configuration Relationship
```
Capstanfile (Image-level)
├─ Specifies base image
├─ Defines build process
├─ Maps application files
└─ Sets boot command

Package Configuration (Package-level)
├─ meta/package.yaml
│  ├─ Package metadata
│  └─ Dependencies
└─ meta/run.yaml
   ├─ Runtime selection
   ├─ Boot configurations
   └─ Environment settings
```

## Repository Ecosystem

### Repository Types

#### 1. GitHub Repository (Default)
- **URL**: https://github.com/cloudius-systems/osv/releases
- **Content**: Official OSv releases and core packages
- **Usage**: `capstan pull <image>` (default source)
- **Versioning**: Release tags (v0.54.0, latest, etc.)

#### 2. S3 Repository (MIKELANGELO)
- **URL**: https://mikelangelo-capstan.s3.amazonaws.com
- **Content**: Extended package collection
- **Usage**: `capstan --s3 pull <image>`
- **Structure**: 
  ```
  mike/
  ├─ osv-loader/
  │  ├─ index.yaml
  │  └─ osv-loader.qemu.gz
  packages/
  ├─ package-name.mpm (compressed package)
  └─ package-name.yaml (metadata)
  ```

#### 3. Local Repository
- **Location**: `~/.capstan/`
- **Structure**:
  ```
  ~/.capstan/
  ├─ repository/          # Local images
  ├─ packages/           # Local packages
  ├─ instances/          # Running instances
  └─ config.yaml         # Capstan configuration
  ```

### Repository Selection Priority
1. Command-line flag (`-u <url>`)
2. Configuration file (`~/.capstan/config.yaml`)
3. Environment variable (`CAPSTAN_REPO_URL`)
4. Default (GitHub repository)

## Next Steps

Now that you understand the overall architecture, you can dive deeper into specific areas:

- **[Comprehensive Workflow Guide](ComprehensiveWorkflow.md)** - Step-by-step process from zero to running application
- **[Image Management](ImageManagement.md)** - Detailed guide on pulling, building, and managing images
- **[Python Complete Guide](PythonComplete.md)** - Comprehensive Python-specific documentation
- **[Multi-Application Scenarios](MultiApplicationScenarios.md)** - Complex deployment patterns
- **[Schema Reference](SchemaReference.md)** - Complete configuration file reference

This architecture provides the foundation for understanding how all OSv and Capstan components work together to create a streamlined unikernel development and deployment experience.