# Getting Started with Capstan and OSv

This comprehensive guide will walk you through everything you need to know to start using Capstan with OSv unikernels, from zero to running your first application.

## Table of Contents

- [What is Capstan?](#what-is-capstan)
- [How Capstan Works](#how-capstan-works)
- [Installation](#installation)
- [Understanding Images and Packages](#understanding-images-and-packages)
- [Your First Unikernel](#your-first-unikernel)
- [Next Steps](#next-steps)

## What is Capstan?

Capstan is a command-line tool that makes it easy to build and run applications on OSv unikernels. Instead of dealing with complex OSv compilation processes, Capstan lets you:

- **Build** unikernel images from precompiled packages in seconds
- **Compose** applications using either Capstanfile (like Dockerfile) or packages (like Docker compose)
- **Run** your unikernel on supported hypervisors (QEMU/KVM, VirtualBox, etc.)

## How Capstan Works

Capstan operates on a simple principle: **compose unikernel images from prebuilt components**.

### The Building Blocks

1. **Base OSv Image (osv-loader)**: The minimal OSv kernel that boots your application
2. **Runtime Packages**: Precompiled language runtimes (Python, Java, Node.js, etc.)
3. **Your Application**: Your code and dependencies
4. **Configuration Files**: Instructions telling Capstan how to run your app

### Two Approaches

**Approach 1: Package-based** (Recommended for most users)
```
Base Image + Runtime Packages + Your App Files = Unikernel Image
```

**Approach 2: Capstanfile-based** (Similar to Dockerfile)
```
Base Image + Build Commands + Files = Unikernel Image
```

## Installation

Follow the [Installation Guide](Installation.md) to install Capstan on your system.

Quick install on Linux:
```bash
curl -O https://github.com/cloudius-systems/capstan/releases/download/v0.2.2/capstan
chmod +x capstan
sudo mv capstan /usr/local/bin/
```

Verify installation:
```bash
capstan --version
```

## Understanding Images and Packages

### Where Do Images Come From?

Capstan can pull images and packages from two sources:

1. **OSv GitHub Releases Repository** (default, recommended)
   - URL: https://github.com/cloudius-systems/osv/releases
   - Contains officially released OSv kernels and packages
   - More up-to-date and actively maintained

2. **S3 Repository** (legacy)
   - URL: https://mikelangelo-capstan.s3.amazonaws.com/
   - Legacy repository from the MIKELANGELO project
   - Use with `--s3` flag

### Pulling Images

**Pull the base OSv loader:**
```bash
# Pull from latest GitHub release
capstan pull cloudius/osv-base

# Pull from specific release
capstan --release-tag v0.54.0 pull cloudius/osv-base
```

**List local images:**
```bash
capstan images
```

Output example:
```
Name                      Description           Version    Created
cloudius/osv-base         OSv Base Image        0.54.0     2019-09-16
myapp                     My Application        -          2024-01-15
```

### Pulling Packages

**Search for available packages:**
```bash
# Search in GitHub releases (default)
capstan --release-tag latest package search

# Search in S3 repository
capstan --s3 package search

# Search for specific package
capstan package search python
```

**Pull specific packages:**
```bash
# Pull from GitHub releases
capstan --release-tag latest package pull osv.python3x

# Pull from S3
capstan --s3 package pull python-2.7
```

**List local packages:**
```bash
capstan package list
```

### Understanding Package Names and Dependencies

Packages can require other packages. For example:
- `osv.httpserver-html5-cli` requires `osv.httpserver-api`
- `osv.httpserver-api` requires `osv.bootstrap`

Capstan automatically resolves and pulls dependencies when composing images.

## Your First Unikernel

Let's create a simple "Hello World" unikernel to understand the workflow.

### Step 1: Create Project Directory

```bash
mkdir my-first-unikernel
cd my-first-unikernel
```

### Step 2: Create Your Application

Create a simple script:
```bash
cat > hello.sh << 'EOF'
#!/bin/sh
echo "Hello from OSv unikernel!"
echo "Current date: $(date)"
EOF
chmod +x hello.sh
```

### Step 3: Initialize Package Metadata

```bash
capstan package init \
  --name "com.example.hello" \
  --title "Hello World Unikernel" \
  --author "Your Name"
```

This creates `meta/package.yaml`:
```yaml
name: com.example.hello
title: Hello World Unikernel
author: Your Name
```

### Step 4: Configure Runtime

```bash
capstan runtime init --runtime native
```

Edit `meta/run.yaml` to run your script:
```yaml
runtime: native

config_set:
  default:
    bootcmd: "/hello.sh"

config_set_default: default
```

### Step 5: Compose the Unikernel Image

```bash
capstan package compose my-hello-app
```

This command:
1. Downloads the base OSv image (if not already present)
2. Uploads your application files
3. Creates a bootable QCOW2 image

### Step 6: Run Your Unikernel

```bash
capstan run my-hello-app
```

You should see output like:
```
OSv v0.54.0
eth0: 192.168.122.15
Hello from OSv unikernel!
Current date: Thu Jan 15 12:34:56 UTC 2024
```

Congratulations! You've created and run your first OSv unikernel.

## Understanding the Workflow

Here's what happens under the hood:

```
1. INITIALIZE
   └─ Create meta/package.yaml and meta/run.yaml

2. COMPOSE
   ├─ Pull base OSv image (cloudius/osv-base)
   ├─ Pull required packages (if any)
   ├─ Create filesystem image
   ├─ Upload your application files
   ├─ Set boot command
   └─ Save as QCOW2 image

3. RUN
   ├─ Start QEMU/KVM
   ├─ Boot OSv kernel
   └─ Execute boot command
```

## Reusing and Building on Images

### Reusing Existing Images

You can build new images based on existing ones:

```bash
# Use existing image as base in Capstanfile
cat > Capstanfile << 'EOF'
base: my-hello-app

cmdline: /new-script.sh

files:
  /new-script.sh: new-script.sh
EOF

capstan build my-enhanced-app
```

### Sharing Images

Export an image:
```bash
capstan export my-hello-app -o my-hello-app.qcow2
```

Import an image:
```bash
capstan import my-hello-app -f my-hello-app.qcow2
```

## Common Configuration Patterns

### Setting Environment Variables

```yaml
runtime: native
config_set:
  default:
    bootcmd: "/app.so"
    env:
      PORT: 8080
      LOG_LEVEL: debug
```

### Multiple Configurations

```yaml
runtime: native

config_set:
  development:
    bootcmd: "/app.so --debug"
    env:
      ENV: dev
  
  production:
    bootcmd: "/app.so"
    env:
      ENV: prod

config_set_default: development
```

Run specific configuration:
```bash
capstan run my-app --runconfig production
```

## Next Steps

Now that you understand the basics, explore:

- **[Image Management Guide](ImageManagement.md)**: Deep dive into image operations
- **[Python Workflow](PythonWorkflow.md)**: Complete guide for Python applications
- **[Running Multiple Jobs](MultipleJobs.md)**: Advanced multi-ELF applications
- **[Configuration Files](ConfigurationFiles.md)**: Detailed configuration reference
- **[Runtime Guides](RuntimePython.md)**: Language-specific guides

### Language-Specific Guides

- [Native Applications](RuntimeNative.md)
- [Python Applications](RuntimePython.md) and [Python Workflow](PythonWorkflow.md)
- [Java Applications](RuntimeJava.md)
- [Node.js Applications](RuntimeNode.md)

### Advanced Topics

- [Capstanfile Reference](Capstanfile.md)
- [Package Management](ApplicationManagement.md)
- [Volumes](Volumes.md)
- [OSv Filesystem](OsvFilesystem.md)

## Troubleshooting

**Image pull fails:**
- Check internet connection
- Try specific release: `capstan --release-tag latest pull cloudius/osv-base`

**Package not found:**
- Search for available packages: `capstan package search`
- Check release tag: `capstan --release-tag latest package search`

**Unikernel crashes on boot:**
- Verify your boot command: `capstan run my-app -v`
- Check runtime compatibility
- Review application logs

## Getting Help

- **CLI Help**: `capstan -h` or `capstan [command] -h`
- **Documentation**: This documentation directory
- **OSv Documentation**: https://github.com/cloudius-systems/osv/wiki
- **Issues**: https://github.com/cloudius-systems/capstan/issues
