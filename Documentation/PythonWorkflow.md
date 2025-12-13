# Complete Python Workflow Guide

This guide provides a comprehensive walkthrough for running Python applications on OSv using Capstan, from zero to production, with focus on Python 3.x (the latest supported version).

## Table of Contents

- [Prerequisites](#prerequisites)
- [Python Support in OSv](#python-support-in-osv)
- [Quick Start](#quick-start)
- [Complete Workflow: From Zero to Running Python](#complete-workflow-from-zero-to-running-python)
- [Python Web Applications](#python-web-applications)
- [Working with Dependencies](#working-with-dependencies)
- [Advanced Configuration](#advanced-configuration)
- [Common Patterns](#common-patterns)
- [Troubleshooting](#troubleshooting)

## Prerequisites

- Capstan installed ([Installation Guide](Installation.md))
- Basic understanding of Python development
- Python 3.x installed locally (for development and testing)

## Python Support in OSv

### Available Python Versions

OSv currently supports:

- **Python 3.x** (Recommended) - via `osv.python3x` package
  - Latest available: Python 3.6+
  - Includes standard library
  - Best compatibility and performance
  
- **Python 2.7** (Deprecated) - via `python-2.7` package
  - Legacy support only
  - Not recommended for new applications

### Pulling Python Runtime

**Python 3.x from GitHub releases (Recommended):**

```bash
# Pull latest Python 3.x package
capstan --release-tag latest package pull osv.python3x
```

**Python 2.7 from S3 (Legacy):**

```bash
# Pull Python 2.7 (not recommended)
capstan --s3 package pull python-2.7
```

### Where Python Comes From

The Python runtime packages are built and published in:

1. **OSv GitHub Releases**: https://github.com/cloudius-systems/osv/releases
   - Pre-compiled Python 3.x binaries
   - Built from OSv apps in `cloudius/python3`
   - Includes Python standard library
   - Regularly updated with OSv releases

2. **S3 Repository** (Legacy): https://mikelangelo-capstan.s3.amazonaws.com/
   - Python 2.7 only
   - No longer maintained

## Quick Start

The fastest way to get Python running:

```bash
# 1. Create project
mkdir my-python-app && cd my-python-app

# 2. Create Python script
cat > app.py << 'EOF'
#!/usr/bin/env python3
print("Hello from Python on OSv!")
import sys
print(f"Python version: {sys.version}")
EOF

# 3. Initialize Capstan configuration
capstan package init --name "com.example.pyapp" --title "My Python App" --author "Your Name"

cat > meta/run.yaml << 'EOF'
runtime: native
config_set:
  default:
    bootcmd: "python3 /app.py"
config_set_default: default
EOF

# 4. Compose and run
capstan package compose my-python-app --pull-missing
capstan run my-python-app
```

## Complete Workflow: From Zero to Running Python

Let's build a complete Python web application step by step.

### Step 1: Set Up Your Python Project Locally

Create a Flask web application:

```bash
# Create project directory
mkdir python-web-app
cd python-web-app

# Create Python virtual environment (for local development)
python3 -m venv venv
source venv/bin/activate

# Install Flask
pip install Flask==2.0.1

# Create requirements.txt
pip freeze > requirements.txt
```

Create `app.py`:

```python
#!/usr/bin/env python3
from flask import Flask, jsonify
import os

app = Flask(__name__)

@app.route('/')
def hello():
    return jsonify({
        'message': 'Hello from Python on OSv!',
        'port': os.environ.get('PORT', 5000)
    })

@app.route('/health')
def health():
    return jsonify({'status': 'healthy'})

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 5000))
    app.run(host='0.0.0.0', port=port)
```

Test locally:
```bash
python app.py
# Visit http://localhost:5000
```

### Step 2: Prepare Dependencies

OSv doesn't have pip, so you need to bundle all Python dependencies.

**Option A: Bundle installed packages**

```bash
# Install all dependencies in a local directory
pip install -r requirements.txt --target ./python-packages

# Directory structure now:
# python-web-app/
# ├── app.py
# ├── requirements.txt
# └── python-packages/
#     ├── flask/
#     ├── werkzeug/
#     └── ...
```

**Option B: Use system Python packages**

```bash
# Copy from your virtual environment
cp -r venv/lib/python3.*/site-packages/* ./python-packages/
```

### Step 3: Configure Capstan

Initialize package metadata:

```bash
capstan package init \
  --name "com.example.flask-app" \
  --title "Flask Web Application" \
  --author "Your Name"
```

Edit `meta/package.yaml` to require Python:

```yaml
name: com.example.flask-app
title: Flask Web Application
author: Your Name
require:
  - osv.python3x
```

Create runtime configuration:

```bash
cat > meta/run.yaml << 'EOF'
runtime: native

config_set:
  default:
    bootcmd: "python3 /app.py"
    env:
      PORT: 5000
      PYTHONPATH: /python-packages
      FLASK_ENV: production

config_set_default: default
EOF
```

### Step 4: Compose the Unikernel

```bash
# Compose the image (automatically pulls osv.python3x if needed)
capstan package compose flask-app --pull-missing -v

# Check the image was created
capstan images
```

### Step 5: Run the Unikernel

```bash
# Run with port forwarding
capstan run flask-app -f 5000:5000
```

In another terminal:
```bash
curl http://localhost:5000
curl http://localhost:5000/health
```

### Step 6: Iterate and Update

Make changes to your application:

```bash
# Edit app.py
nano app.py

# Update the image (fast - only uploads changed files)
capstan package compose flask-app --update

# Run again
capstan run flask-app -f 5000:5000
```

## Python Web Applications

### Flask Application

Complete Flask setup:

```yaml
# meta/package.yaml
name: com.example.flask-app
title: Flask App
author: Your Name
require:
  - osv.python3x
```

```yaml
# meta/run.yaml
runtime: native

config_set:
  web:
    bootcmd: "python3 /app.py"
    env:
      PORT: 8000
      PYTHONPATH: /python-packages
      FLASK_APP: app.py
      FLASK_ENV: production

config_set_default: web
```

### Django Application

```yaml
# meta/run.yaml
runtime: native

config_set:
  web:
    bootcmd: "python3 /manage.py runserver 0.0.0.0:8000"
    env:
      PYTHONPATH: /python-packages
      DJANGO_SETTINGS_MODULE: myproject.settings

config_set_default: web
```

### FastAPI Application

```python
# app.py
from fastapi import FastAPI
import uvicorn
import os

app = FastAPI()

@app.get("/")
def read_root():
    return {"message": "Hello from FastAPI on OSv"}

if __name__ == "__main__":
    port = int(os.environ.get('PORT', 8000))
    uvicorn.run(app, host="0.0.0.0", port=port)
```

```yaml
# meta/run.yaml
runtime: native

config_set:
  api:
    bootcmd: "python3 /app.py"
    env:
      PORT: 8000
      PYTHONPATH: /python-packages

config_set_default: api
```

## Working with Dependencies

### Managing Python Packages

**Strategy 1: Bundle everything**

```bash
# Install all dependencies locally
pip install -r requirements.txt --target ./packages

# Set PYTHONPATH in run.yaml
env:
  PYTHONPATH: /packages
```

**Strategy 2: Create a package directory structure**

```bash
# Create proper Python package structure
mkdir -p lib/python3/site-packages
pip install -r requirements.txt --target ./lib/python3/site-packages

# Use standard Python path
env:
  PYTHONPATH: /lib/python3/site-packages
```

**Strategy 3: Vendor dependencies**

```bash
# Copy dependencies into your app directory
mkdir -p myapp/vendor
pip install -r requirements.txt --target ./myapp/vendor

# In your app.py
import sys
sys.path.insert(0, '/myapp/vendor')
```

### Working with C Extensions

Some Python packages include C extensions (numpy, scipy, etc.). These need special handling:

```bash
# Option 1: Build on compatible platform (Ubuntu 14.04/16.04)
docker run -v $(pwd):/build ubuntu:16.04 bash -c \
  "apt-get update && apt-get install -y python3-pip && \
   pip3 install numpy --target /build/packages"

# Option 2: Use pre-built wheels for compatible platforms
pip install numpy --platform manylinux2014_x86_64 --target ./packages
```

### Example with Multiple Dependencies

```bash
# requirements.txt
Flask==2.0.1
requests==2.26.0
redis==3.5.3

# Install
pip install -r requirements.txt --target ./packages

# Project structure
python-app/
├── app.py
├── packages/
│   ├── flask/
│   ├── requests/
│   └── redis/
└── meta/
    ├── package.yaml
    └── run.yaml
```

## Advanced Configuration

### Multiple Named Configurations

Run the same app in different modes:

```yaml
# meta/run.yaml
runtime: native

config_set:
  development:
    bootcmd: "python3 /app.py"
    env:
      PORT: 5000
      FLASK_ENV: development
      DEBUG: "1"
      PYTHONPATH: /packages
  
  production:
    bootcmd: "python3 /app.py"
    env:
      PORT: 8000
      FLASK_ENV: production
      DEBUG: "0"
      PYTHONPATH: /packages
  
  worker:
    bootcmd: "python3 /worker.py"
    env:
      PYTHONPATH: /packages
      WORKER_THREADS: "4"

config_set_default: production
```

Usage:
```bash
# Run in development mode
capstan run my-app --runconfig development

# Run in production mode
capstan run my-app --runconfig production

# Run as worker
capstan run my-app --runconfig worker
```

### Interactive Python Shell

```yaml
# meta/run.yaml
runtime: native

config_set:
  shell:
    bootcmd: "python3"
    env:
      PYTHONPATH: /packages

config_set_default: shell
```

Run:
```bash
capstan run python-shell --runconfig shell
# You get an interactive Python prompt
```

### Running Python Scripts with Arguments

```yaml
# meta/run.yaml
runtime: native

config_set:
  script:
    bootcmd: "python3 /script.py --input /data/input.csv --output /data/output.csv"
    env:
      PYTHONPATH: /packages

config_set_default: script
```

### Environment-Specific Configuration

```yaml
# meta/run.yaml
runtime: native

config_set:
  default:
    bootcmd: "python3 /app.py"
    env:
      PYTHONPATH: /packages
      DATABASE_URL: "postgresql://db:5432/myapp"
      REDIS_URL: "redis://redis:6379/0"
      SECRET_KEY: "change-me-in-production"
      LOG_LEVEL: "INFO"

config_set_default: default
```

## Common Patterns

### Pattern 1: Simple Script Execution

**Use Case**: Run a Python script once and exit

```yaml
runtime: native
config_set:
  default:
    bootcmd: "python3 /process_data.py"
config_set_default: default
```

### Pattern 2: Long-Running Web Service

**Use Case**: Run a web server continuously

```yaml
runtime: native
config_set:
  default:
    bootcmd: "python3 /server.py"
    env:
      PORT: 8000
      PYTHONPATH: /packages
config_set_default: default
```

### Pattern 3: Scheduled Task / Worker

**Use Case**: Background worker processing tasks

```python
# worker.py
import time
import os

def process_task():
    print("Processing task...")
    time.sleep(int(os.environ.get('INTERVAL', 10)))

if __name__ == '__main__':
    while True:
        process_task()
```

```yaml
runtime: native
config_set:
  worker:
    bootcmd: "python3 /worker.py"
    env:
      INTERVAL: "10"
      PYTHONPATH: /packages
config_set_default: worker
```

### Pattern 4: Multiple Python Files

**Project Structure:**
```
my-app/
├── app.py
├── utils.py
├── models.py
├── config.py
└── packages/
```

```python
# app.py
from utils import helper_function
from models import User
from config import get_config

def main():
    config = get_config()
    # ... your code
```

No special configuration needed - Python imports work normally!

### Pattern 5: Data Processing Pipeline

```python
# pipeline.py
import sys
import json

def process(input_file, output_file):
    with open(input_file) as f:
        data = json.load(f)
    
    # Process data
    result = transform(data)
    
    with open(output_file, 'w') as f:
        json.dump(result, f)

if __name__ == '__main__':
    process(sys.argv[1], sys.argv[2])
```

```yaml
runtime: native
config_set:
  default:
    bootcmd: "python3 /pipeline.py /data/input.json /data/output.json"
config_set_default: default
```

## Troubleshooting

### Python Module Not Found

**Problem**: `ModuleNotFoundError: No module named 'flask'`

**Solution**: 
```bash
# Ensure dependencies are bundled
pip install -r requirements.txt --target ./packages

# Set PYTHONPATH in run.yaml
env:
  PYTHONPATH: /packages
```

### Import Errors with C Extensions

**Problem**: `ImportError: cannot import name '_ssl'`

**Solution**: Build on compatible platform or use compatible wheels:
```bash
# Use compatible manylinux wheels
pip install --platform manylinux2014_x86_64 --target ./packages <package>
```

### Python Version Mismatch

**Problem**: Code works locally but fails in OSv

**Solution**: Check Python version compatibility:
```bash
# In your local Python
python3 --version

# Use compatible syntax
# Avoid Python 3.9+ features if OSv has Python 3.6
```

### Application Exits Immediately

**Problem**: Unikernel boots and exits

**Solution**: Ensure your application keeps running:
```python
# Wrong - exits immediately
print("Hello")

# Right - keeps running
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello():
    return "Hello"

if __name__ == '__main__':
    app.run(host='0.0.0.0')  # Blocks and keeps running
```

### Port Already in Use

**Problem**: `Address already in use`

**Solution**: 
```bash
# Use different host port
capstan run my-app -f 5001:5000

# Or stop conflicting service
capstan instances
capstan stop <instance-name>
```

### Slow Compose Time

**Problem**: Composing takes too long

**Solution**: Use update flag for iterations:
```bash
# First time
capstan package compose my-app

# Subsequent times after code changes
capstan package compose my-app --update
```

### Cannot Connect to Application

**Problem**: Unikernel runs but can't access web app

**Solution**: 
1. Ensure port forwarding: `capstan run my-app -f 8000:8000`
2. Check app binds to `0.0.0.0`, not `127.0.0.1`:
```python
# Wrong
app.run(host='127.0.0.1', port=5000)

# Right
app.run(host='0.0.0.0', port=5000)
```

## Best Practices

1. **Always specify Python 3**: Use `python3` explicitly in bootcmd
2. **Bundle dependencies**: Include all packages in your image
3. **Use PYTHONPATH**: Set environment variable for clean imports
4. **Test locally first**: Ensure code works before composing
5. **Use --update flag**: Speed up iterations during development
6. **Set environment variables**: Configure app via env vars in run.yaml
7. **Keep images small**: Only include necessary dependencies
8. **Pin versions**: Use specific versions in requirements.txt

## Next Steps

- Learn about [Multiple Jobs](MultipleJobs.md) for running multiple Python processes
- Explore [Image Management](ImageManagement.md) for advanced image operations
- See [Volumes](Volumes.md) for persistent data storage
- Review [Configuration Files](ConfigurationFiles.md) for all options

## Reference

### Required Files for Python Applications

```
my-python-app/
├── app.py                 # Your Python application
├── packages/              # Python dependencies
│   └── ...
└── meta/
    ├── package.yaml       # Package metadata
    └── run.yaml          # Runtime configuration
```

### Minimal run.yaml for Python

```yaml
runtime: native
config_set:
  default:
    bootcmd: "python3 /app.py"
    env:
      PYTHONPATH: /packages
config_set_default: default
```

### Common Environment Variables

- `PYTHONPATH`: Colon-separated list of directories to search for modules
- `PYTHONHOME`: Python installation directory
- `PYTHONDONTWRITEBYTECODE`: Set to any value to prevent .pyc files
- `PYTHONUNBUFFERED`: Set to any value for unbuffered output
