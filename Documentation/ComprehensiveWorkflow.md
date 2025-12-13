# Complete Workflow: From Zero to Running Python on OSv

This guide provides a comprehensive, step-by-step workflow for getting from a fresh system to running Python applications on OSv using Capstan. This is the definitive guide for beginners and a reference for experienced users.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Step 1: Install Capstan](#step-1-install-capstan)
- [Step 2: Understand the OSv Base Image](#step-2-understand-the-osv-base-image)
- [Step 3: Your First Python Application](#step-3-your-first-python-application)
- [Step 4: Using Packages (Recommended Method)](#step-4-using-packages-recommended-method)
- [Step 5: Using Capstanfile (Docker-like Method)](#step-5-using-capstanfile-docker-like-method)
- [Step 6: Advanced Configuration](#step-6-advanced-configuration)
- [Step 7: Running and Testing](#step-7-running-and-testing)
- [Troubleshooting](#troubleshooting)
- [Next Steps](#next-steps)

## Prerequisites

### System Requirements
- **Linux**: Ubuntu 18.04+, CentOS 7+, or similar
- **macOS**: 10.14+ (with limitations)
- **Windows**: WSL2 recommended
- **Memory**: At least 4GB RAM
- **Disk Space**: 2GB free space
- **Virtualization**: KVM support (Linux) or VirtualBox/VMware

### Required Software
```bash
# Ubuntu/Debian
sudo apt update
sudo apt install qemu-kvm qemu-utils libvirt-daemon-system libvirt-clients bridge-utils

# CentOS/RHEL
sudo yum install qemu-kvm qemu-img libvirt libvirt-python libvirt-client virt-install virt-viewer bridge-utils

# macOS (using Homebrew)
brew install qemu
```

### Verify Virtualization Support
```bash
# Linux - Check KVM support
lscpu | grep Virtualization
sudo kvm-ok  # Should show "KVM acceleration can be used"

# Check if your user is in the kvm group
groups $USER | grep kvm
# If not, add yourself: sudo usermod -a -G kvm $USER
```

## Step 1: Install Capstan

### Option A: Download Precompiled Binary (Recommended)
```bash
# Download latest release
curl -L https://github.com/cloudius-systems/capstan/releases/latest/download/capstan -o capstan
chmod +x capstan
sudo mv capstan /usr/local/bin/

# Verify installation
capstan --version
```

### Option B: Build from Source
```bash
# Install Go 1.16+
# Ubuntu/Debian
sudo apt install golang-go

# Clone and build
git clone https://github.com/cloudius-systems/capstan.git
cd capstan
go build
sudo mv capstan /usr/local/bin/
```

### Initial Configuration
```bash
# Create Capstan configuration directory
mkdir -p ~/.capstan

# Optional: Configure default repository
cat > ~/.capstan/config.yaml << EOF
repo_url: ""  # Uses GitHub by default
disable_kvm: false
qemu_aio_type: threads
release_tag: latest
EOF
```

## Step 2: Understand the OSv Base Image

### What is a Base Image?
A base image is a minimal OSv kernel that serves as the foundation for your application. Think of it as a minimal Linux distribution, but much smaller and faster.

### Available Base Images
```bash
# List available images from GitHub repository
capstan search

# Common base images:
# - osv-loader: Minimal OSv kernel (smallest)
# - osv-base: OSv with basic utilities
# - cloudius/osv-base: Community base image
```

### Pull Your First Base Image
```bash
# Pull the standard base image
capstan pull cloudius/osv-base

# Verify it was downloaded
capstan images
```

**Expected Output:**
```
NAME                SIZE      CREATED
cloudius/osv-base   ~50MB     2023-XX-XX
```

## Step 3: Your First Python Application

### Create a Simple Python Application
```bash
# Create project directory
mkdir my-python-app
cd my-python-app

# Create a simple Python script
cat > hello.py << 'EOF'
#!/usr/bin/env python3
import sys
import os

print("Hello from OSv!")
print(f"Python version: {sys.version}")
print(f"Current directory: {os.getcwd()}")
print(f"Arguments: {sys.argv}")

# Simple web server example
if len(sys.argv) > 1 and sys.argv[1] == "server":
    from http.server import HTTPServer, SimpleHTTPRequestHandler
    import socketserver
    
    PORT = 8000
    Handler = SimpleHTTPRequestHandler
    
    with socketserver.TCPServer(("", PORT), Handler) as httpd:
        print(f"Server running at port {PORT}")
        httpd.serve_forever()
EOF
```

### Test Locally First
```bash
# Always test your application locally before packaging
python3 hello.py
python3 hello.py server  # Test web server (Ctrl+C to stop)
```

## Step 4: Using Packages (Recommended Method)

This is the modern, recommended approach using Capstan's package system.

### Initialize Package Structure
```bash
# Initialize package metadata
capstan package init --name "my-python-app" --title "My First Python App" --author "Your Name"

# Initialize Python runtime configuration
capstan runtime init --runtime native

# Verify structure
tree .
```

**Expected Structure:**
```
my-python-app/
├── hello.py
└── meta/
    ├── package.yaml
    └── run.yaml
```

### Configure Package Dependencies
Edit `meta/package.yaml`:
```yaml
name: my-python-app
title: My First Python App
author: Your Name
require:
  - osv.python3x  # Latest Python 3.x runtime
```

### Configure Runtime
Edit `meta/run.yaml`:
```yaml
runtime: native

config_set:
  default:
    bootcmd: python3 /hello.py
  server:
    bootcmd: python3 /hello.py server
    env:
      PYTHONPATH: /
  debug:
    bootcmd: python3 -u /hello.py
    env:
      PYTHONPATH: /
      PYTHONUNBUFFERED: "1"

config_set_default: default
```

### Build and Compose the Image
```bash
# Compose the package into a bootable image
capstan package compose my-python-app --pull-missing

# Verify the image was created
capstan images
```

**Expected Output:**
```
NAME            SIZE      CREATED
my-python-app   ~120MB    2023-XX-XX (just now)
```

### Run Your Application
```bash
# Run with default configuration
capstan run my-python-app

# Run with server configuration
capstan run my-python-app --boot server

# Run with debug configuration
capstan run my-python-app --boot debug
```

**Expected Output:**
```
Created instance: my-python-app
Setting cmdline: python3 /hello.py
OSv v0.54.0
eth0: 192.168.122.15
Hello from OSv!
Python version: 3.8.10 (default, ...)
Current directory: /
Arguments: ['/hello.py']
```

## Step 5: Using Capstanfile (Docker-like Method)

This method uses a Capstanfile similar to Docker's approach.

### Create a New Project
```bash
mkdir my-capstanfile-app
cd my-capstanfile-app

# Create the same Python application
cat > app.py << 'EOF'
#!/usr/bin/env python3
print("Hello from Capstanfile!")

import json
import sys

config = {
    "message": "Running on OSv via Capstanfile",
    "python_version": sys.version,
    "arguments": sys.argv
}

print(json.dumps(config, indent=2))
EOF
```

### Create Capstanfile
```yaml
# Capstanfile
base: cloudius/osv-base

cmdline: /usr/bin/python3 /app/app.py

files:
  /app/app.py: app.py
  /usr/bin/python3: /usr/bin/python3  # If not in base image
```

### Build with Capstanfile
```bash
# Build the image
capstan build my-capstanfile-app

# Run the image
capstan run my-capstanfile-app
```

## Step 6: Advanced Configuration

### Working with Dependencies
If your Python application has dependencies:

```bash
# Create requirements.txt
cat > requirements.txt << 'EOF'
requests==2.28.1
flask==2.2.2
gunicorn==20.1.0
EOF

# Create a more complex application
cat > web_app.py << 'EOF'
#!/usr/bin/env python3
from flask import Flask, jsonify
import os

app = Flask(__name__)

@app.route('/')
def hello():
    return jsonify({
        "message": "Hello from OSv + Flask!",
        "environment": dict(os.environ),
        "python_path": os.sys.path
    })

@app.route('/health')
def health():
    return jsonify({"status": "healthy"})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080, debug=True)
EOF
```

### Update Package Configuration
Edit `meta/run.yaml` for the Flask app:
```yaml
runtime: native

config_set:
  flask-dev:
    bootcmd: python3 /web_app.py
    env:
      FLASK_ENV: development
      PYTHONPATH: /
  flask-prod:
    bootcmd: gunicorn --bind 0.0.0.0:8080 web_app:app
    env:
      FLASK_ENV: production
      PYTHONPATH: /

config_set_default: flask-dev
```

### Install Dependencies in Image
For applications with pip dependencies, you'll need to create a custom base image or use the build process:

```yaml
# Enhanced Capstanfile with dependency installation
base: cloudius/osv-base

build: |
  pip3 install -r requirements.txt --target /app/lib
  
cmdline: PYTHONPATH=/app/lib python3 /app/web_app.py

files:
  /app/web_app.py: web_app.py
  /app/requirements.txt: requirements.txt
```

## Step 7: Running and Testing

### Basic Running
```bash
# Run with default settings
capstan run my-python-app

# Run with specific configuration
capstan run my-python-app --boot server

# Run with custom command line
capstan run my-python-app -e "python3 /hello.py custom_arg"
```

### Networking and Port Forwarding
```bash
# Forward port 8080 from guest to host
capstan run my-python-app --boot server -f 8080:8080

# Test the connection
curl http://localhost:8080
```

### Debugging and Logs
```bash
# Run with verbose output
capstan run my-python-app --boot debug -v

# Run with custom memory size
capstan run my-python-app -m 1G

# Run and attach to console
capstan run my-python-app --boot debug
# Press Ctrl+A, then X to exit QEMU console
```

### Managing Instances
```bash
# List running instances
capstan instances

# Stop an instance
capstan stop my-python-app

# Delete an instance
capstan delete my-python-app
```

## Troubleshooting

### Common Issues and Solutions

#### 1. "KVM not available" Error
```bash
# Check KVM support
sudo kvm-ok

# If KVM is not available, disable it
capstan run my-python-app --disable-kvm
```

#### 2. "Package not found" Error
```bash
# Pull missing packages
capstan package pull osv.python3x

# Or use --pull-missing flag
capstan package compose my-python-app --pull-missing
```

#### 3. Python Import Errors
```bash
# Check Python path in your run.yaml
env:
  PYTHONPATH: /:/app:/app/lib
```

#### 4. Network Issues
```bash
# Check if the application is binding to the right interface
# Use 0.0.0.0 instead of localhost or 127.0.0.1
app.run(host='0.0.0.0', port=8080)
```

#### 5. Memory Issues
```bash
# Increase memory allocation
capstan run my-python-app -m 2G
```

### Debugging Steps
1. **Test locally first**: Always verify your Python application works on your host system
2. **Check logs**: Use `capstan run -v` for verbose output
3. **Verify packages**: Use `capstan package list` to ensure required packages are available
4. **Check configuration**: Verify your `meta/run.yaml` and `meta/package.yaml` files
5. **Network debugging**: Use `capstan run -f 8080:8080` to forward ports for web applications

## Next Steps

Congratulations! You now have a working Python application running on OSv. Here's what you can explore next:

### Immediate Next Steps
- **[Python Complete Guide](PythonComplete.md)** - Deep dive into Python-specific features and optimizations
- **[Image Management](ImageManagement.md)** - Learn about pulling, building, and managing images
- **[Multi-Application Scenarios](MultiApplicationScenarios.md)** - Run multiple applications or services

### Advanced Topics
- **Performance Optimization**: Tune your unikernel for production workloads
- **Custom Base Images**: Create your own base images with pre-installed dependencies
- **CI/CD Integration**: Integrate Capstan into your deployment pipeline
- **Monitoring and Logging**: Set up monitoring for your OSv applications

### Production Considerations
- **Security**: Review security implications of running in kernel space
- **Scaling**: Understand how to scale unikernel applications
- **Persistence**: Learn about data persistence strategies
- **Networking**: Advanced networking configurations for production

You now have the foundation to build and deploy Python applications on OSv using Capstan. The key is to start simple and gradually add complexity as you become more comfortable with the platform.