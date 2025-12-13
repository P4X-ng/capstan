# Runtime `python`
This document describes how to write a valid `meta/run.yaml` configuration file
for running **Python** applications on OSv.

## Modern Python 3.x (Recommended)

**⚠️ Important**: Python 2.7 is deprecated and no longer supported. All new projects should use Python 3.x.

### Required Package
```yaml
# meta/package.yaml
require:
  - osv.python3x  # Python 3.8.10 (latest supported)
```

### Basic Configuration
```yaml
# meta/run.yaml
runtime: native

config_set:
  default:
    bootcmd: python3 /app/main.py
    env:
      PYTHONPATH: /app:/app/lib
      PYTHONUNBUFFERED: "1"
```

### Interactive Python Shell
```yaml
# meta/run.yaml
runtime: native

config_set:
  shell:
    bootcmd: python3 -i
    env:
      PYTHONPATH: /app:/app/lib
```

**Example**:
```bash
$ capstan package compose my-python-app --pull-missing
$ capstan run my-python-app --boot shell
Created instance: my-python-app
Setting cmdline: python3 -i
OSv v0.54.0
eth0: 192.168.122.15
Python 3.8.10 (default, Nov 14 2022, 12:59:47)
[GCC 9.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>>
```

## Python Application Examples

### Simple Python Script
```python
# main.py
#!/usr/bin/env python3
import sys
import os

def main():
    print("Hello from OSv!")
    print(f"Python version: {sys.version}")
    print(f"Arguments: {sys.argv[1:]}")
    print(f"Environment: {dict(os.environ)}")

if __name__ == '__main__':
    main()
```

**Configuration**:
```yaml
# meta/run.yaml
runtime: native

config_set:
  default:
    bootcmd: python3 /app/main.py
    env:
      PYTHONPATH: /app:/app/lib
      
  with-args:
    bootcmd: python3 /app/main.py arg1 arg2
    env:
      PYTHONPATH: /app:/app/lib
```

### Web Application (Flask)
```python
# web_app.py
#!/usr/bin/env python3
from flask import Flask, jsonify
import os

app = Flask(__name__)

@app.route('/')
def hello():
    return jsonify({
        "message": "Hello from OSv + Python 3!",
        "python_version": os.sys.version,
        "environment": dict(os.environ)
    })

@app.route('/health')
def health():
    return jsonify({"status": "healthy"})

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 8080))
    app.run(host='0.0.0.0', port=port, debug=False)
```

**Configuration**:
```yaml
# meta/run.yaml
runtime: native

config_set:
  web-server:
    bootcmd: python3 /app/web_app.py
    env:
      PYTHONPATH: /app:/app/lib
      PORT: "8080"
      FLASK_ENV: production
      
  debug:
    bootcmd: python3 /app/web_app.py
    env:
      PYTHONPATH: /app:/app/lib
      PORT: "8080"
      FLASK_ENV: development
      DEBUG: "true"

config_set_default: web-server
```

**Usage**:
```bash
$ capstan package compose flask-app --pull-missing
$ capstan run flask-app --boot web-server -f 8080:8080
# Test: curl http://localhost:8080/
```

### Multiple Configuration Example
```yaml
# meta/run.yaml - Complete example
runtime: native

config_set:
  web-server:
    bootcmd: python3 /app/server.py
    env:
      PYTHONPATH: /app:/app/lib
      PORT: "8080"
      MODE: "server"
      
  worker:
    bootcmd: python3 /app/worker.py
    env:
      PYTHONPATH: /app:/app/lib
      MODE: "worker"
      
  migrate:
    bootcmd: python3 /app/manage.py migrate
    env:
      PYTHONPATH: /app:/app/lib
      MODE: "migrate"
      
  test:
    bootcmd: python3 -m pytest /app/tests/ -v
    env:
      PYTHONPATH: /app:/app/lib:/app/tests
      TESTING: "true"
      
  shell:
    bootcmd: python3 -i
    env:
      PYTHONPATH: /app:/app/lib

config_set_default: web-server
```

## Legacy Python 2.7 (Deprecated)

**⚠️ Warning**: Python 2.7 support is deprecated and should not be used for new projects.

### Legacy Configuration (Not Recommended)
```yaml
# meta/package.yaml - DEPRECATED
require:
  - python-2.7  # Deprecated package

# meta/run.yaml - DEPRECATED
runtime: python

config_set:
  legacy-app:
    main: /app/script.py
    args:
      - arg1
      - arg2
```

## Migration from Python 2.7 to Python 3.x

If you have existing Python 2.7 applications, migrate them using this guide:

### 1. Update Package Dependencies
```yaml
# OLD (Python 2.7)
require:
  - python-2.7

# NEW (Python 3.x)
require:
  - osv.python3x
```

### 2. Update Runtime Configuration
```yaml
# OLD (Python 2.7)
runtime: python
config_set:
  default:
    main: /app/script.py

# NEW (Python 3.x)
runtime: native
config_set:
  default:
    bootcmd: python3 /app/script.py
    env:
      PYTHONPATH: /app:/app/lib
```

### 3. Update Python Code
```python
# OLD (Python 2.7)
print 'Hello World'
import ConfigParser

# NEW (Python 3.x)
print('Hello World')
import configparser
```

## Best Practices

### 1. Always Use Python 3.x
```yaml
require:
  - osv.python3x  # Latest supported Python 3.x
```

### 2. Use Native Runtime
```yaml
runtime: native  # More flexible than legacy python runtime
```

### 3. Set PYTHONPATH
```yaml
env:
  PYTHONPATH: /app:/app/lib  # Ensure modules can be found
```

### 4. Use Unbuffered Output
```yaml
env:
  PYTHONUNBUFFERED: "1"  # Immediate stdout/stderr output
```

### 5. Environment-Specific Configurations
```yaml
config_set:
  development:
    bootcmd: python3 -u /app/main.py
    env:
      DEBUG: "true"
      
  production:
    bootcmd: python3 -O /app/main.py
    env:
      DEBUG: "false"
      PYTHONOPTIMIZE: "2"
```

For more comprehensive Python documentation, see [Python Complete Guide](PythonComplete.md).
