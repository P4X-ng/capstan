# Runtime `python`
This document describes how to write a valid `meta/run.yaml` configuration file
for running **Python** applications.

## Python 3.x Support (Recommended)

OSv supports **Python 3.x** (Python 3.6+) through the `osv.python3x` package, which is the recommended and actively maintained Python runtime.

### Quick Start with Python 3

**1. Pull the Python 3 package:**
```bash
capstan --release-tag latest package pull osv.python3x
```

**2. Require python in your package:**
```yaml
# meta/package.yaml
name: my-python-app
require:
  - osv.python3x
```

**3. Configure the runtime:**
```yaml
# meta/run.yaml
runtime: native

config_set: 
   default:
      bootcmd: "python3 /app.py"
      env:
         PYTHONPATH: /packages

config_set_default: default
```

**4. Compose and run:**
```bash
capstan package compose my-app --pull-missing
capstan run my-app
```

### Where Python Comes From

Python packages are available from:
- **OSv GitHub Releases** (recommended): https://github.com/cloudius-systems/osv/releases
  - Package: `osv.python3x`
  - Python version: 3.6+
  - Actively maintained
  
- **S3 Repository** (legacy): For Python 2.7 only (deprecated)

### Python 3 Features

The `osv.python3x` package includes:
- Python 3.6+ interpreter
- Standard library modules
- Support for C extensions (with compatible compilation)
- Threading support (use threads, not fork)

For a comprehensive guide on Python development with OSv, see the [Python Workflow Guide](PythonWorkflow.md).

## Running Python Scripts

### Simple Script Execution

Run a Python 3 script with arguments:

```yaml
# meta/run.yaml
runtime: native

config_set:
  default:
    bootcmd: "python3 /script.py arg1 arg2"
    env:
      PYTHONPATH: /packages

config_set_default: default
```

Example script:
```python
#!/usr/bin/env python3
import sys

print("Hello from Python 3!")
print("Arguments:")
for arg in sys.argv[1:]:
    print(f"  - {arg}")
```

### Interactive Python Shell

To run an interactive Python 3 interpreter:

```yaml
# meta/run.yaml
runtime: native

config_set:
  shell:
    bootcmd: "python3"

config_set_default: shell
```

Run:
```bash
$ capstan package compose python-shell
$ capstan run python-shell --boot shell
OSv v0.54.0
eth0: 192.168.122.15
Python 3.6.9 (default, Jan 01 2020, 00:00:00)
[GCC 7.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>>
```

### Python Web Application

Run a Flask/Django web server:

```yaml
# meta/run.yaml
runtime: native

config_set:
  web:
    bootcmd: "python3 /app.py"
    env:
      PORT: 8000
      PYTHONPATH: /packages
      FLASK_ENV: production

config_set_default: web
```

Example Flask app:
```python
#!/usr/bin/env python3
from flask import Flask
import os

app = Flask(__name__)

@app.route('/')
def hello():
    return "Hello from Python 3 on OSv!"

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 8000))
    app.run(host='0.0.0.0', port=port)
```

## Working with Python Dependencies

Since OSv doesn't have pip, bundle all dependencies:

```bash
# Install dependencies locally
pip install -r requirements.txt --target ./packages

# Set PYTHONPATH in run.yaml
env:
  PYTHONPATH: /packages
```

See the [Python Workflow Guide](PythonWorkflow.md) for comprehensive examples.

## Multiple Python Jobs

To run multiple Python applications concurrently, use threading or runscripts:

```yaml
# meta/run.yaml
runtime: native

config_set:
  multi:
    bootcmd: "runscript /init.yaml"
    env:
      PYTHONPATH: /packages

config_set_default: multi
```

Runscript `/init.yaml`:
```
python3 /worker1.py &
python3 /worker2.py &
python3 /server.py
```

For detailed multi-job patterns, see [Multiple Jobs Guide](MultipleJobs.md).

---

## Python 2.7 (Deprecated - Legacy Only)

**Note**: Python 2.7 is deprecated and no longer maintained. Use Python 3.x for new applications.

For legacy applications, Python 2.7 is available from S3 repository:

```bash
# Pull Python 2.7 (not recommended)
capstan --s3 package pull python-2.7
```

The `python` runtime (deprecated) can be used with Python 2.7:

```yaml
# meta/run.yaml
runtime: python

config_set:
  default:
    main: /script.py
    args:
      - arg1
```

However, migrating to Python 3 with the `native` runtime is strongly recommended.

---

## Next Steps

- **[Python Workflow Guide](PythonWorkflow.md)**: Complete guide from zero to production
- **[Multiple Jobs Guide](MultipleJobs.md)**: Running multiple Python processes
- **[Image Management](ImageManagement.md)**: Managing Python packages and images
- **[Configuration Files](ConfigurationFiles.md)**: All configuration options
