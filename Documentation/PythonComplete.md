# Complete Python Guide for OSv

This comprehensive guide covers everything you need to know about running Python applications on OSv using Capstan, including the latest supported Python versions, modern Python 3.x configuration, and best practices.

## Table of Contents
- [Python Versions and Availability](#python-versions-and-availability)
- [Modern Python 3.x Configuration](#modern-python-3x-configuration)
- [Package Management](#package-management)
- [Framework Integration](#framework-integration)
- [Performance Optimization](#performance-optimization)
- [Development Workflow](#development-workflow)
- [Production Deployment](#production-deployment)
- [Troubleshooting](#troubleshooting)

## Python Versions and Availability

### Currently Supported Python Versions

**Latest Supported**: Python 3.8.10 (as of OSv v0.54.0)

**Available Python Packages**:
```bash
# Check available Python packages
capstan --s3 package search python

# Common Python packages:
osv.python3x        # Python 3.8.10 (recommended)
osv.python38        # Python 3.8.x specific
python-2.7          # Python 2.7 (deprecated, legacy only)
```

**Verification**:
```bash
# Pull and inspect Python package
capstan --s3 package pull osv.python3x
capstan package describe osv.python3x
```

### Python 3.x Features Available
- **Full Standard Library**: Complete Python 3.8 standard library
- **pip Package Manager**: Install additional packages
- **C Extensions**: Support for compiled extensions
- **Threading**: Python threading support (within OSv limitations)
- **Networking**: Full socket and HTTP client/server support
- **File I/O**: Complete file system operations

### Deprecated Python Versions
⚠️ **Python 2.7 is deprecated** and should not be used for new projects:
```yaml
# DON'T USE - Deprecated
require:
  - python-2.7  # Legacy only, no longer maintained
```

## Modern Python 3.x Configuration

### Recommended Method: Native Runtime

**Modern Approach** (Recommended):
```yaml
# meta/package.yaml
name: my-python-app
title: Modern Python Application
author: Your Name
require:
  - osv.python3x

# meta/run.yaml
runtime: native

config_set:
  default:
    bootcmd: python3 /app/main.py
    env:
      PYTHONPATH: /app:/app/lib
      PYTHONUNBUFFERED: "1"
```

### Legacy Method: Python Runtime

**Legacy Approach** (Not recommended for new projects):
```yaml
# meta/run.yaml - Legacy method
runtime: python

config_set:
  default:
    main: /app/main.py
    args:
      - --verbose
```

### Configuration Best Practices

#### Environment Variables
```yaml
# meta/run.yaml
runtime: native

config_set:
  development:
    bootcmd: python3 -u /app/main.py
    env:
      PYTHONPATH: /app:/app/lib
      PYTHONUNBUFFERED: "1"      # Immediate stdout/stderr
      PYTHONDONTWRITEBYTECODE: "1" # Don't create .pyc files
      FLASK_ENV: development
      DEBUG: "true"
      
  production:
    bootcmd: python3 -O /app/main.py
    env:
      PYTHONPATH: /app:/app/lib
      PYTHONUNBUFFERED: "1"
      FLASK_ENV: production
      DEBUG: "false"
      
  testing:
    bootcmd: python3 -m pytest /app/tests/
    env:
      PYTHONPATH: /app:/app/lib:/app/tests
      PYTEST_CURRENT_TEST: "1"
```

#### Multiple Entry Points
```yaml
# meta/run.yaml - Multiple configurations
runtime: native

config_set:
  web-server:
    bootcmd: python3 /app/server.py
    env:
      PYTHONPATH: /app
      PORT: "8080"
      
  worker:
    bootcmd: python3 /app/worker.py
    env:
      PYTHONPATH: /app
      WORKER_TYPE: background
      
  migrate:
    bootcmd: python3 /app/manage.py migrate
    env:
      PYTHONPATH: /app
      
  shell:
    bootcmd: python3 -i /app/shell.py
    env:
      PYTHONPATH: /app

config_set_default: web-server
```

## Package Management

### Installing Python Packages

#### Method 1: Pre-install in Build Process
```yaml
# Capstanfile
base: cloudius/osv-base

build: |
  # Install packages to specific directory
  pip3 install -r requirements.txt --target /app/lib --no-cache-dir
  
  # Install development tools if needed
  pip3 install pytest flake8 --target /app/dev-lib --no-cache-dir
  
  # Clean up
  rm -rf /tmp/* /var/tmp/*

cmdline: python3 /app/main.py

files:
  /app/main.py: src/main.py
  /app/requirements.txt: requirements.txt
```

#### Method 2: Package-based Installation
```bash
# Create requirements package
mkdir python-deps
cd python-deps

# Create package structure
capstan package init --name python-deps --title "Python Dependencies"

# Install dependencies during package build
cat > build.sh << 'EOF'
#!/bin/bash
pip3 install -r requirements.txt --target /app/lib --no-cache-dir
EOF

chmod +x build.sh
./build.sh

# Build and import package
capstan package build
capstan package import
```

#### Method 3: Runtime Installation (Not Recommended)
```yaml
# Not recommended for production - slower boot times
runtime: native
config_set:
  default:
    bootcmd: |
      pip3 install flask gunicorn &&
      python3 /app/main.py
```

### Managing Dependencies

#### requirements.txt Best Practices
```txt
# requirements.txt - Pin versions for reproducibility
flask==2.2.2
gunicorn==20.1.0
requests==2.28.1
psycopg2-binary==2.9.3
redis==4.3.4

# Development dependencies (separate file)
# requirements-dev.txt
pytest==7.1.2
flake8==5.0.4
black==22.6.0
```

#### Handling C Extensions
```yaml
# For packages with C extensions
build: |
  # Install build tools first
  apt-get update && apt-get install -y gcc python3-dev
  
  # Install packages
  pip3 install numpy pandas --target /app/lib --no-cache-dir
  
  # Clean up build tools
  apt-get remove -y gcc python3-dev
  apt-get autoremove -y
```

## Framework Integration

### Flask Applications

#### Simple Flask App
```python
# app/main.py
from flask import Flask, jsonify, request
import os

app = Flask(__name__)

@app.route('/')
def hello():
    return jsonify({
        "message": "Hello from OSv!",
        "python_version": os.sys.version,
        "environment": dict(os.environ)
    })

@app.route('/health')
def health():
    return jsonify({"status": "healthy"})

@app.route('/api/data', methods=['GET', 'POST'])
def api_data():
    if request.method == 'POST':
        return jsonify({"received": request.get_json()})
    return jsonify({"data": "sample data"})

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 8080))
    debug = os.environ.get('DEBUG', 'false').lower() == 'true'
    app.run(host='0.0.0.0', port=port, debug=debug)
```

#### Flask Configuration
```yaml
# meta/run.yaml
runtime: native

config_set:
  flask-dev:
    bootcmd: python3 /app/main.py
    env:
      PYTHONPATH: /app:/app/lib
      FLASK_ENV: development
      DEBUG: "true"
      PORT: "8080"
      
  flask-prod:
    bootcmd: gunicorn --bind 0.0.0.0:8080 --workers 1 main:app
    env:
      PYTHONPATH: /app:/app/lib
      FLASK_ENV: production
      DEBUG: "false"
      PORT: "8080"

config_set_default: flask-dev
```

### Django Applications

#### Django Project Structure
```
my-django-app/
├── manage.py
├── myproject/
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── myapp/
│   ├── __init__.py
│   ├── models.py
│   ├── views.py
│   └── urls.py
├── requirements.txt
└── meta/
    ├── package.yaml
    └── run.yaml
```

#### Django Configuration
```yaml
# meta/run.yaml
runtime: native

config_set:
  runserver:
    bootcmd: python3 /app/manage.py runserver 0.0.0.0:8080
    env:
      PYTHONPATH: /app:/app/lib
      DJANGO_SETTINGS_MODULE: myproject.settings
      
  migrate:
    bootcmd: python3 /app/manage.py migrate
    env:
      PYTHONPATH: /app:/app/lib
      DJANGO_SETTINGS_MODULE: myproject.settings
      
  collectstatic:
    bootcmd: python3 /app/manage.py collectstatic --noinput
    env:
      PYTHONPATH: /app:/app/lib
      DJANGO_SETTINGS_MODULE: myproject.settings

config_set_default: runserver
```

### FastAPI Applications

#### FastAPI Example
```python
# app/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import uvicorn
import os

app = FastAPI(title="OSv FastAPI App", version="1.0.0")

class Item(BaseModel):
    name: str
    description: str = None
    price: float

@app.get("/")
async def root():
    return {"message": "FastAPI running on OSv"}

@app.get("/health")
async def health_check():
    return {"status": "healthy"}

@app.post("/items/")
async def create_item(item: Item):
    return {"item": item, "message": "Item created"}

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 8080))
    uvicorn.run(app, host="0.0.0.0", port=port)
```

#### FastAPI Configuration
```yaml
# meta/run.yaml
runtime: native

config_set:
  fastapi-dev:
    bootcmd: python3 /app/main.py
    env:
      PYTHONPATH: /app:/app/lib
      PORT: "8080"
      
  fastapi-prod:
    bootcmd: uvicorn main:app --host 0.0.0.0 --port 8080
    env:
      PYTHONPATH: /app:/app/lib
      PORT: "8080"

config_set_default: fastapi-dev
```

### Data Science Applications

#### Jupyter Notebook Server
```python
# app/notebook_server.py
import os
import subprocess
import sys

def start_jupyter():
    # Configure Jupyter
    os.environ['JUPYTER_CONFIG_DIR'] = '/app/jupyter_config'
    
    # Start Jupyter notebook server
    cmd = [
        sys.executable, '-m', 'jupyter', 'notebook',
        '--ip=0.0.0.0',
        '--port=8888',
        '--no-browser',
        '--allow-root',
        '--notebook-dir=/app/notebooks'
    ]
    
    subprocess.run(cmd)

if __name__ == '__main__':
    start_jupyter()
```

#### Data Science Configuration
```yaml
# meta/run.yaml
runtime: native

config_set:
  jupyter:
    bootcmd: python3 /app/notebook_server.py
    env:
      PYTHONPATH: /app:/app/lib
      JUPYTER_CONFIG_DIR: /app/jupyter_config
      
  analysis:
    bootcmd: python3 /app/analysis.py
    env:
      PYTHONPATH: /app:/app/lib
      MATPLOTLIB_BACKEND: Agg  # Non-interactive backend

config_set_default: jupyter
```

## Performance Optimization

### Memory Management
```yaml
# Optimize Python memory usage
config_set:
  optimized:
    bootcmd: python3 -O /app/main.py
    env:
      PYTHONPATH: /app:/app/lib
      PYTHONUNBUFFERED: "1"
      PYTHONDONTWRITEBYTECODE: "1"  # Don't create .pyc files
      PYTHONHASHSEED: "0"           # Deterministic hash seed
```

### Startup Optimization
```python
# app/optimized_main.py
import sys
import os

# Optimize imports - import only what you need
def main():
    # Lazy imports for faster startup
    from flask import Flask
    
    app = Flask(__name__)
    
    @app.route('/')
    def hello():
        return "Hello from optimized OSv app!"
    
    port = int(os.environ.get('PORT', 8080))
    app.run(host='0.0.0.0', port=port, debug=False)

if __name__ == '__main__':
    main()
```

### Production Optimizations
```yaml
# meta/run.yaml - Production optimized
runtime: native

config_set:
  production:
    bootcmd: python3 -O -u /app/main.py
    env:
      PYTHONPATH: /app:/app/lib
      PYTHONUNBUFFERED: "1"
      PYTHONDONTWRITEBYTECODE: "1"
      PYTHONOPTIMIZE: "2"           # Remove docstrings and assertions
      MALLOC_TRIM_THRESHOLD_: "65536"  # Memory optimization
```

## Development Workflow

### Local Development Setup
```bash
# 1. Create project structure
mkdir my-python-osv-app
cd my-python-osv-app

# 2. Create virtual environment for local development
python3 -m venv venv
source venv/bin/activate

# 3. Install dependencies locally
pip install flask gunicorn pytest

# 4. Create requirements.txt
pip freeze > requirements.txt

# 5. Initialize Capstan package
capstan package init --name my-python-osv-app --title "My Python OSv App"
capstan runtime init --runtime native
```

### Development Cycle
```bash
# 1. Develop and test locally
python3 app/main.py

# 2. Build OSv image
capstan package compose my-python-osv-app --pull-missing

# 3. Test in OSv
capstan run my-python-osv-app --boot development

# 4. Debug if needed
capstan run my-python-osv-app --boot debug -v

# 5. Test production configuration
capstan run my-python-osv-app --boot production
```

### Testing Strategy
```python
# tests/test_main.py
import pytest
import sys
import os

# Add app directory to path
sys.path.insert(0, os.path.join(os.path.dirname(__file__), '..', 'app'))

from main import app

@pytest.fixture
def client():
    app.config['TESTING'] = True
    with app.test_client() as client:
        yield client

def test_hello_endpoint(client):
    response = client.get('/')
    assert response.status_code == 200
    assert b'Hello from OSv' in response.data

def test_health_endpoint(client):
    response = client.get('/health')
    assert response.status_code == 200
    data = response.get_json()
    assert data['status'] == 'healthy'
```

### Testing Configuration
```yaml
# meta/run.yaml - Include testing configuration
config_set:
  test:
    bootcmd: python3 -m pytest /app/tests/ -v
    env:
      PYTHONPATH: /app:/app/lib:/app/tests
      PYTEST_CURRENT_TEST: "1"
      
  coverage:
    bootcmd: python3 -m pytest /app/tests/ --cov=/app --cov-report=term-missing
    env:
      PYTHONPATH: /app:/app/lib:/app/tests
```

## Production Deployment

### Production-Ready Configuration
```yaml
# meta/run.yaml - Production setup
runtime: native

config_set:
  production:
    bootcmd: gunicorn --bind 0.0.0.0:8080 --workers 1 --timeout 30 --keep-alive 2 main:app
    env:
      PYTHONPATH: /app:/app/lib
      FLASK_ENV: production
      GUNICORN_WORKERS: "1"
      GUNICORN_TIMEOUT: "30"
      GUNICORN_KEEPALIVE: "2"
      PYTHONUNBUFFERED: "1"
      PYTHONDONTWRITEBYTECODE: "1"
      
  production-debug:
    bootcmd: gunicorn --bind 0.0.0.0:8080 --workers 1 --log-level debug main:app
    env:
      PYTHONPATH: /app:/app/lib
      FLASK_ENV: production
      GUNICORN_LOG_LEVEL: debug

config_set_default: production
```

### Health Checks and Monitoring
```python
# app/health.py
import psutil
import time
from flask import jsonify

def get_system_health():
    """Get system health metrics"""
    return {
        "status": "healthy",
        "timestamp": time.time(),
        "memory": {
            "available": psutil.virtual_memory().available,
            "percent": psutil.virtual_memory().percent
        },
        "cpu": {
            "percent": psutil.cpu_percent(interval=1)
        },
        "uptime": time.time() - psutil.boot_time()
    }

# Add to your main Flask app
@app.route('/health/detailed')
def detailed_health():
    try:
        health_data = get_system_health()
        return jsonify(health_data)
    except Exception as e:
        return jsonify({"status": "unhealthy", "error": str(e)}), 500
```

### Logging Configuration
```python
# app/logging_config.py
import logging
import sys
import os

def setup_logging():
    """Configure logging for OSv environment"""
    log_level = os.environ.get('LOG_LEVEL', 'INFO').upper()
    
    # Configure root logger
    logging.basicConfig(
        level=getattr(logging, log_level),
        format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
        stream=sys.stdout  # OSv works better with stdout
    )
    
    # Configure specific loggers
    logging.getLogger('werkzeug').setLevel(logging.WARNING)
    logging.getLogger('urllib3').setLevel(logging.WARNING)
    
    return logging.getLogger(__name__)

# Use in your main application
logger = setup_logging()
logger.info("Application starting on OSv")
```

### Environment-Specific Builds
```yaml
# Capstanfile.production
base: cloudius/osv-base

build: |
  # Production optimizations
  pip3 install -r requirements.txt --target /app/lib --no-cache-dir --no-deps
  
  # Remove development files
  find /app -name "*.pyc" -delete
  find /app -name "__pycache__" -type d -exec rm -rf {} +
  
  # Set production permissions
  chmod -R 755 /app

cmdline: python3 -O /app/main.py

files:
  /app/main.py: src/main.py
  /app/requirements.txt: requirements.txt
  /app/config/: config/production/
```

## Troubleshooting

### Common Issues and Solutions

#### 1. Import Errors
```bash
# Problem: ModuleNotFoundError
# Solution: Check PYTHONPATH
config_set:
  debug:
    bootcmd: python3 -c "import sys; print('\\n'.join(sys.path))"
    env:
      PYTHONPATH: /app:/app/lib
```

#### 2. Package Installation Issues
```bash
# Problem: pip install fails during build
# Solution: Use --no-cache-dir and --target
build: |
  pip3 install --no-cache-dir --target /app/lib -r requirements.txt
  
  # For packages with C extensions
  pip3 install --no-cache-dir --target /app/lib --no-binary=:all: some-package
```

#### 3. Network Binding Issues
```python
# Problem: Application not accessible from host
# Solution: Bind to 0.0.0.0, not localhost
if __name__ == '__main__':
    # WRONG: app.run(host='localhost', port=8080)
    # CORRECT:
    app.run(host='0.0.0.0', port=8080)
```

#### 4. File Path Issues
```python
# Problem: File not found errors
# Solution: Use absolute paths in OSv
import os

# Get absolute path relative to application
app_dir = os.path.dirname(os.path.abspath(__file__))
config_path = os.path.join(app_dir, 'config.json')

# Or use environment variables
config_path = os.environ.get('CONFIG_PATH', '/app/config.json')
```

#### 5. Memory Issues
```yaml
# Problem: Out of memory errors
# Solution: Increase memory allocation
# Run with more memory:
# capstan run my-app -m 2G

# Or optimize Python memory usage
config_set:
  memory-optimized:
    bootcmd: python3 -O /app/main.py
    env:
      PYTHONHASHSEED: "0"
      PYTHONDONTWRITEBYTECODE: "1"
```

### Debugging Techniques

#### 1. Verbose Logging
```yaml
config_set:
  debug:
    bootcmd: python3 -u -v /app/main.py
    env:
      PYTHONPATH: /app:/app/lib
      PYTHONUNBUFFERED: "1"
      PYTHONVERBOSE: "1"
      DEBUG: "true"
```

#### 2. Interactive Debugging
```python
# app/debug_main.py
import code
import sys

def debug_shell():
    """Start interactive Python shell for debugging"""
    code.interact(local=locals())

if __name__ == '__main__':
    if '--debug-shell' in sys.argv:
        debug_shell()
    else:
        # Normal application startup
        from main import app
        app.run(host='0.0.0.0', port=8080, debug=True)
```

#### 3. Environment Inspection
```python
# app/inspect_env.py
import os
import sys
import json

def inspect_environment():
    """Inspect OSv environment for debugging"""
    info = {
        "python_version": sys.version,
        "python_path": sys.path,
        "environment_variables": dict(os.environ),
        "current_directory": os.getcwd(),
        "available_modules": list(sys.modules.keys())
    }
    
    print(json.dumps(info, indent=2, default=str))

if __name__ == '__main__':
    inspect_environment()
```

### Performance Troubleshooting

#### 1. Startup Time Issues
```bash
# Measure startup time
time capstan run my-app -e "python3 -c 'print(\"Started\")'"

# Profile startup
capstan run my-app -e "python3 -X importtime /app/main.py"
```

#### 2. Memory Usage Analysis
```python
# app/memory_profiler.py
import tracemalloc
import psutil
import os

def profile_memory():
    """Profile memory usage"""
    tracemalloc.start()
    
    # Your application code here
    from main import app
    
    current, peak = tracemalloc.get_traced_memory()
    process = psutil.Process(os.getpid())
    
    print(f"Current memory usage: {current / 1024 / 1024:.1f} MB")
    print(f"Peak memory usage: {peak / 1024 / 1024:.1f} MB")
    print(f"Process memory: {process.memory_info().rss / 1024 / 1024:.1f} MB")
    
    tracemalloc.stop()

if __name__ == '__main__':
    profile_memory()
```

### Best Practices Summary

1. **Always use Python 3.x** - Python 2.7 is deprecated
2. **Use native runtime** - More flexible than legacy python runtime
3. **Pin dependency versions** - Ensure reproducible builds
4. **Test locally first** - Verify your application works before packaging
5. **Use absolute paths** - OSv filesystem structure may differ from expectations
6. **Bind to 0.0.0.0** - Not localhost for network services
7. **Set PYTHONUNBUFFERED=1** - For immediate output in logs
8. **Use --pull-missing** - Automatically pull required packages
9. **Optimize for production** - Use -O flag and remove debug code
10. **Monitor resource usage** - OSv has different resource characteristics

This comprehensive guide provides everything you need to successfully develop, deploy, and troubleshoot Python applications on OSv using Capstan. The key is to understand the differences between traditional Linux environments and the OSv unikernel environment, and adapt your development practices accordingly.