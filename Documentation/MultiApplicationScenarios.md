# Multi-Application Scenarios: Multi-ELF and Multi-Job OSv Deployments

This guide covers advanced deployment patterns for running multiple applications, services, or jobs within OSv unikernels using Capstan. Learn how to configure complex scenarios with Python, Java, Perl, and other runtimes.

## Table of Contents
- [Understanding Multi-Application Patterns](#understanding-multi-application-patterns)
- [Multi-ELF Configurations](#multi-elf-configurations)
- [Multi-Job Scenarios](#multi-job-scenarios)
- [Python Multi-Application Examples](#python-multi-application-examples)
- [Java Multi-Application Examples](#java-multi-application-examples)
- [Mixed Runtime Scenarios](#mixed-runtime-scenarios)
- [Advanced Capstan Configurations](#advanced-capstan-configurations)
- [Best Practices](#best-practices)

## Understanding Multi-Application Patterns

### OSv Single-Process Limitation
OSv is designed to run **one primary process** per unikernel instance. However, you can achieve multi-application functionality through several patterns:

1. **Multi-ELF**: Multiple executable files with different entry points
2. **Multi-Job**: Single application managing multiple tasks/workers
3. **Service Orchestration**: Multiple related services in one image
4. **Runtime Switching**: Different behaviors based on configuration

### When to Use Multi-Application Patterns

**Use Cases**:
- Microservices that need to share resources
- Applications with multiple operational modes (web server, worker, migration)
- Development environments with multiple tools
- Applications requiring different startup configurations

**Alternatives to Consider**:
- Multiple separate unikernel instances (recommended for true isolation)
- Container orchestration for complex multi-service applications
- Traditional VM deployment for legacy multi-process applications

## Multi-ELF Configurations

### Pattern 1: Multiple Python Scripts

Create an image with multiple Python applications:

```bash
# Project structure
multi-python-app/
├── web_server.py
├── data_processor.py
├── admin_tool.py
├── requirements.txt
└── meta/
    ├── package.yaml
    └── run.yaml
```

#### Application Files
```python
# web_server.py
from flask import Flask, jsonify
import os

app = Flask(__name__)

@app.route('/')
def hello():
    return jsonify({"service": "web-server", "status": "running"})

@app.route('/health')
def health():
    return jsonify({"status": "healthy"})

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 8080))
    app.run(host='0.0.0.0', port=port)
```

```python
# data_processor.py
import time
import json
import sys

def process_data():
    """Simulate data processing"""
    print("Starting data processing...")
    
    for i in range(10):
        print(f"Processing batch {i+1}/10")
        time.sleep(1)
    
    print("Data processing completed")
    return {"status": "completed", "batches_processed": 10}

if __name__ == '__main__':
    result = process_data()
    print(json.dumps(result))
```

```python
# admin_tool.py
import sys
import json

def show_status():
    return {"tool": "admin", "version": "1.0", "status": "ready"}

def migrate_data():
    print("Running data migration...")
    # Simulate migration
    return {"migration": "completed"}

def main():
    if len(sys.argv) < 2:
        print("Usage: python3 admin_tool.py <command>")
        print("Commands: status, migrate")
        sys.exit(1)
    
    command = sys.argv[1]
    
    if command == "status":
        result = show_status()
    elif command == "migrate":
        result = migrate_data()
    else:
        print(f"Unknown command: {command}")
        sys.exit(1)
    
    print(json.dumps(result))

if __name__ == '__main__':
    main()
```

#### Multi-ELF Configuration
```yaml
# meta/package.yaml
name: multi-python-app
title: Multi-Python Application Suite
author: Your Name
require:
  - osv.python3x
  - osv.httpserver

# meta/run.yaml
runtime: native

config_set:
  web-server:
    bootcmd: python3 /web_server.py
    env:
      PYTHONPATH: /:/lib
      PORT: "8080"
      SERVICE_MODE: web
      
  data-processor:
    bootcmd: python3 /data_processor.py
    env:
      PYTHONPATH: /:/lib
      SERVICE_MODE: processor
      
  admin-status:
    bootcmd: python3 /admin_tool.py status
    env:
      PYTHONPATH: /:/lib
      SERVICE_MODE: admin
      
  admin-migrate:
    bootcmd: python3 /admin_tool.py migrate
    env:
      PYTHONPATH: /:/lib
      SERVICE_MODE: admin

config_set_default: web-server
```

#### Usage
```bash
# Build the multi-application image
capstan package compose multi-python-app --pull-missing

# Run different applications
capstan run multi-python-app --boot web-server
capstan run multi-python-app --boot data-processor
capstan run multi-python-app --boot admin-status
capstan run multi-python-app --boot admin-migrate
```

### Pattern 2: Application Launcher

Create a launcher script that can start different applications:

```python
# launcher.py
import sys
import os
import subprocess

APPLICATIONS = {
    'web': {
        'cmd': ['python3', '/web_server.py'],
        'env': {'PORT': '8080', 'MODE': 'web'}
    },
    'worker': {
        'cmd': ['python3', '/data_processor.py'],
        'env': {'MODE': 'worker'}
    },
    'admin': {
        'cmd': ['python3', '/admin_tool.py'],
        'env': {'MODE': 'admin'}
    }
}

def main():
    if len(sys.argv) < 2:
        print("Available applications:", list(APPLICATIONS.keys()))
        sys.exit(1)
    
    app_name = sys.argv[1]
    app_args = sys.argv[2:] if len(sys.argv) > 2 else []
    
    if app_name not in APPLICATIONS:
        print(f"Unknown application: {app_name}")
        sys.exit(1)
    
    app_config = APPLICATIONS[app_name]
    
    # Set environment variables
    env = os.environ.copy()
    env.update(app_config['env'])
    
    # Execute the application
    cmd = app_config['cmd'] + app_args
    subprocess.run(cmd, env=env)

if __name__ == '__main__':
    main()
```

```yaml
# meta/run.yaml with launcher
runtime: native

config_set:
  web:
    bootcmd: python3 /launcher.py web
    env:
      PYTHONPATH: /:/lib
      
  worker:
    bootcmd: python3 /launcher.py worker
    env:
      PYTHONPATH: /:/lib
      
  admin:
    bootcmd: python3 /launcher.py admin status
    env:
      PYTHONPATH: /:/lib

config_set_default: web
```

## Multi-Job Scenarios

### Pattern 1: Multi-Threaded Python Application

```python
# multi_job_app.py
import threading
import time
import queue
import signal
import sys
from flask import Flask, jsonify
import logging

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class MultiJobApplication:
    def __init__(self):
        self.app = Flask(__name__)
        self.job_queue = queue.Queue()
        self.workers = []
        self.running = True
        
        # Setup Flask routes
        self.setup_routes()
        
        # Setup signal handlers
        signal.signal(signal.SIGTERM, self.shutdown)
        signal.signal(signal.SIGINT, self.shutdown)
    
    def setup_routes(self):
        @self.app.route('/')
        def status():
            return jsonify({
                "status": "running",
                "active_workers": len([w for w in self.workers if w.is_alive()]),
                "queue_size": self.job_queue.qsize()
            })
        
        @self.app.route('/submit/<job_type>')
        def submit_job(job_type):
            job_id = f"{job_type}_{int(time.time())}"
            self.job_queue.put({"id": job_id, "type": job_type})
            return jsonify({"job_id": job_id, "status": "queued"})
    
    def worker_thread(self, worker_id):
        """Worker thread that processes jobs"""
        logger.info(f"Worker {worker_id} started")
        
        while self.running:
            try:
                job = self.job_queue.get(timeout=1)
                logger.info(f"Worker {worker_id} processing job {job['id']}")
                
                # Simulate job processing
                time.sleep(2)
                
                logger.info(f"Worker {worker_id} completed job {job['id']}")
                self.job_queue.task_done()
                
            except queue.Empty:
                continue
            except Exception as e:
                logger.error(f"Worker {worker_id} error: {e}")
        
        logger.info(f"Worker {worker_id} stopped")
    
    def start_workers(self, num_workers=3):
        """Start worker threads"""
        for i in range(num_workers):
            worker = threading.Thread(target=self.worker_thread, args=(i,))
            worker.daemon = True
            worker.start()
            self.workers.append(worker)
        
        logger.info(f"Started {num_workers} worker threads")
    
    def shutdown(self, signum, frame):
        """Graceful shutdown"""
        logger.info("Shutting down...")
        self.running = False
        
        # Wait for workers to finish
        for worker in self.workers:
            worker.join(timeout=5)
        
        sys.exit(0)
    
    def run(self):
        """Start the application"""
        # Start worker threads
        self.start_workers()
        
        # Start Flask web server
        port = int(os.environ.get('PORT', 8080))
        self.app.run(host='0.0.0.0', port=port, threaded=True)

if __name__ == '__main__':
    import os
    app = MultiJobApplication()
    app.run()
```

#### Configuration for Multi-Job App
```yaml
# meta/run.yaml
runtime: native

config_set:
  multi-job:
    bootcmd: python3 /multi_job_app.py
    env:
      PYTHONPATH: /:/lib
      PORT: "8080"
      WORKERS: "3"
      
  single-job:
    bootcmd: python3 /multi_job_app.py
    env:
      PYTHONPATH: /:/lib
      PORT: "8080"
      WORKERS: "1"

config_set_default: multi-job
```

### Pattern 2: Async Python Application

```python
# async_multi_job.py
import asyncio
import aiohttp
from aiohttp import web
import signal
import logging
import json
import time

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class AsyncMultiJobApp:
    def __init__(self):
        self.app = web.Application()
        self.jobs = {}
        self.running = True
        
        # Setup routes
        self.app.router.add_get('/', self.status)
        self.app.router.add_post('/jobs', self.submit_job)
        self.app.router.add_get('/jobs/{job_id}', self.get_job_status)
        
        # Setup signal handlers
        signal.signal(signal.SIGTERM, self.shutdown)
        signal.signal(signal.SIGINT, self.shutdown)
    
    async def status(self, request):
        return web.json_response({
            "status": "running",
            "active_jobs": len([j for j in self.jobs.values() if j['status'] == 'running']),
            "total_jobs": len(self.jobs)
        })
    
    async def submit_job(self, request):
        data = await request.json()
        job_id = f"job_{int(time.time())}"
        
        # Start job as background task
        task = asyncio.create_task(self.process_job(job_id, data))
        
        self.jobs[job_id] = {
            "id": job_id,
            "status": "running",
            "data": data,
            "task": task
        }
        
        return web.json_response({"job_id": job_id, "status": "submitted"})
    
    async def get_job_status(self, request):
        job_id = request.match_info['job_id']
        
        if job_id not in self.jobs:
            return web.json_response({"error": "Job not found"}, status=404)
        
        job = self.jobs[job_id]
        return web.json_response({
            "job_id": job_id,
            "status": job["status"],
            "result": job.get("result")
        })
    
    async def process_job(self, job_id, data):
        """Process a job asynchronously"""
        try:
            logger.info(f"Processing job {job_id}")
            
            # Simulate async work
            await asyncio.sleep(5)
            
            result = {"processed": True, "data": data}
            
            self.jobs[job_id]["status"] = "completed"
            self.jobs[job_id]["result"] = result
            
            logger.info(f"Completed job {job_id}")
            
        except Exception as e:
            logger.error(f"Job {job_id} failed: {e}")
            self.jobs[job_id]["status"] = "failed"
            self.jobs[job_id]["error"] = str(e)
    
    def shutdown(self, signum, frame):
        """Graceful shutdown"""
        logger.info("Shutting down...")
        self.running = False
        
        # Cancel all running tasks
        for job in self.jobs.values():
            if "task" in job and not job["task"].done():
                job["task"].cancel()
    
    async def run(self):
        """Start the application"""
        port = int(os.environ.get('PORT', 8080))
        
        runner = web.AppRunner(self.app)
        await runner.setup()
        
        site = web.TCPSite(runner, '0.0.0.0', port)
        await site.start()
        
        logger.info(f"Server started on port {port}")
        
        # Keep running
        try:
            while self.running:
                await asyncio.sleep(1)
        except KeyboardInterrupt:
            pass
        finally:
            await runner.cleanup()

if __name__ == '__main__':
    import os
    app = AsyncMultiJobApp()
    asyncio.run(app.run())
```

## Java Multi-Application Examples

### Pattern 1: Java Multi-Main Classes

```java
// WebServer.java
import com.sun.net.httpserver.HttpServer;
import com.sun.net.httpserver.HttpHandler;
import com.sun.net.httpserver.HttpExchange;
import java.io.IOException;
import java.io.OutputStream;
import java.net.InetSocketAddress;

public class WebServer {
    public static void main(String[] args) throws IOException {
        int port = Integer.parseInt(System.getProperty("port", "8080"));
        
        HttpServer server = HttpServer.create(new InetSocketAddress(port), 0);
        
        server.createContext("/", new HttpHandler() {
            @Override
            public void handle(HttpExchange exchange) throws IOException {
                String response = "{\"service\":\"java-web\",\"status\":\"running\"}";
                exchange.sendResponseHeaders(200, response.length());
                OutputStream os = exchange.getResponseBody();
                os.write(response.getBytes());
                os.close();
            }
        });
        
        server.setExecutor(null);
        server.start();
        
        System.out.println("Java web server started on port " + port);
    }
}
```

```java
// DataProcessor.java
import java.util.concurrent.TimeUnit;

public class DataProcessor {
    public static void main(String[] args) {
        System.out.println("Starting Java data processor...");
        
        for (int i = 1; i <= 10; i++) {
            System.out.println("Processing batch " + i + "/10");
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
        
        System.out.println("Data processing completed");
    }
}
```

#### Java Multi-Application Configuration
```yaml
# meta/package.yaml
name: multi-java-app
title: Multi-Java Application Suite
author: Your Name
require:
  - osv.java

# meta/run.yaml
runtime: native

config_set:
  web-server:
    bootcmd: java -cp /app WebServer
    env:
      JAVA_OPTS: "-Xmx512m"
      port: "8080"
      
  data-processor:
    bootcmd: java -cp /app DataProcessor
    env:
      JAVA_OPTS: "-Xmx256m"
      
  both-sequential:
    bootcmd: |
      java -cp /app DataProcessor &&
      java -cp /app WebServer
    env:
      JAVA_OPTS: "-Xmx512m"

config_set_default: web-server
```

### Pattern 2: Java Application Launcher

```java
// ApplicationLauncher.java
import java.lang.reflect.Method;
import java.util.HashMap;
import java.util.Map;

public class ApplicationLauncher {
    private static final Map<String, String> APPLICATIONS = new HashMap<>();
    
    static {
        APPLICATIONS.put("web", "WebServer");
        APPLICATIONS.put("processor", "DataProcessor");
        APPLICATIONS.put("admin", "AdminTool");
    }
    
    public static void main(String[] args) {
        if (args.length < 1) {
            System.out.println("Available applications: " + APPLICATIONS.keySet());
            System.exit(1);
        }
        
        String appName = args[0];
        String className = APPLICATIONS.get(appName);
        
        if (className == null) {
            System.out.println("Unknown application: " + appName);
            System.exit(1);
        }
        
        try {
            Class<?> appClass = Class.forName(className);
            Method mainMethod = appClass.getMethod("main", String[].class);
            
            // Pass remaining arguments to the application
            String[] appArgs = new String[args.length - 1];
            System.arraycopy(args, 1, appArgs, 0, appArgs.length);
            
            mainMethod.invoke(null, (Object) appArgs);
            
        } catch (Exception e) {
            System.err.println("Failed to launch application: " + e.getMessage());
            e.printStackTrace();
            System.exit(1);
        }
    }
}
```

## Mixed Runtime Scenarios

### Pattern 1: Python + Java Integration

Create an image that can run both Python and Java applications:

```yaml
# meta/package.yaml
name: mixed-runtime-app
title: Mixed Python and Java Application
author: Your Name
require:
  - osv.python3x
  - osv.java
  - osv.httpserver

# meta/run.yaml
runtime: native

config_set:
  python-web:
    bootcmd: python3 /app/web_server.py
    env:
      PYTHONPATH: /app:/app/lib
      PORT: "8080"
      
  java-web:
    bootcmd: java -cp /app WebServer
    env:
      JAVA_OPTS: "-Xmx512m"
      
  python-processor:
    bootcmd: python3 /app/data_processor.py
    env:
      PYTHONPATH: /app:/app/lib
      
  java-processor:
    bootcmd: java -cp /app DataProcessor
    env:
      JAVA_OPTS: "-Xmx256m"
      
  mixed-pipeline:
    bootcmd: |
      python3 /app/data_processor.py > /tmp/processed_data.json &&
      java -cp /app DataAnalyzer /tmp/processed_data.json
    env:
      PYTHONPATH: /app:/app/lib
      JAVA_OPTS: "-Xmx512m"

config_set_default: python-web
```

### Pattern 2: Perl Integration

```perl
#!/usr/bin/perl
# web_server.pl
use strict;
use warnings;
use HTTP::Server::Simple::CGI;

package MyWebServer;
use base qw(HTTP::Server::Simple::CGI);

sub handle_request {
    my ($self, $cgi) = @_;
    
    print "HTTP/1.0 200 OK\r\n";
    print "Content-Type: application/json\r\n\r\n";
    print '{"service":"perl-web","status":"running"}';
}

package main;

my $port = $ENV{PORT} || 8080;
my $server = MyWebServer->new($port);
print "Perl web server starting on port $port\n";
$server->run();
```

```yaml
# Mixed runtime with Perl
config_set:
  perl-web:
    bootcmd: perl /app/web_server.pl
    env:
      PORT: "8080"
      PERL5LIB: /app/lib
      
  python-perl-pipeline:
    bootcmd: |
      python3 /app/generate_data.py | perl /app/process_data.pl > /app/results.txt &&
      cat /app/results.txt
    env:
      PYTHONPATH: /app:/app/lib
      PERL5LIB: /app/lib
```

## Advanced Capstan Configurations

### Pattern 1: Environment-Based Application Selection

```yaml
# meta/run.yaml - Environment-driven configuration
runtime: native

config_set:
  auto-detect:
    bootcmd: |
      if [ "$APP_TYPE" = "web" ]; then
        python3 /app/web_server.py
      elif [ "$APP_TYPE" = "worker" ]; then
        python3 /app/worker.py
      elif [ "$APP_TYPE" = "admin" ]; then
        python3 /app/admin.py
      else
        echo "Unknown APP_TYPE: $APP_TYPE"
        exit 1
      fi
    env:
      PYTHONPATH: /app:/app/lib
      APP_TYPE: web  # Default
      
  web:
    bootcmd: python3 /app/web_server.py
    env:
      PYTHONPATH: /app:/app/lib
      APP_TYPE: web
      
  worker:
    bootcmd: python3 /app/worker.py
    env:
      PYTHONPATH: /app:/app/lib
      APP_TYPE: worker

config_set_default: auto-detect
```

### Pattern 2: Complex Initialization Scripts

```bash
#!/bin/bash
# init_script.sh
set -e

echo "Starting multi-application initialization..."

# Set up environment
export PYTHONPATH="/app:/app/lib"
export JAVA_OPTS="-Xmx512m"

# Check configuration
if [ -z "$SERVICE_MODE" ]; then
    echo "SERVICE_MODE not set, defaulting to web"
    export SERVICE_MODE="web"
fi

# Initialize based on mode
case "$SERVICE_MODE" in
    "web")
        echo "Starting web service..."
        python3 /app/web_server.py
        ;;
    "worker")
        echo "Starting worker service..."
        python3 /app/worker.py
        ;;
    "hybrid")
        echo "Starting hybrid service..."
        python3 /app/hybrid_app.py
        ;;
    "java-web")
        echo "Starting Java web service..."
        java -cp /app WebServer
        ;;
    *)
        echo "Unknown service mode: $SERVICE_MODE"
        exit 1
        ;;
esac
```

```yaml
# meta/run.yaml using initialization script
runtime: native

config_set:
  web:
    bootcmd: /app/init_script.sh
    env:
      SERVICE_MODE: web
      
  worker:
    bootcmd: /app/init_script.sh
    env:
      SERVICE_MODE: worker
      
  hybrid:
    bootcmd: /app/init_script.sh
    env:
      SERVICE_MODE: hybrid
      
  java-web:
    bootcmd: /app/init_script.sh
    env:
      SERVICE_MODE: java-web

config_set_default: web
```

## Best Practices

### 1. Resource Management
```yaml
# Consider memory allocation for multi-application scenarios
config_set:
  memory-optimized:
    bootcmd: python3 -O /app/main.py
    env:
      PYTHONOPTIMIZE: "2"
      MALLOC_TRIM_THRESHOLD_: "65536"
```

### 2. Logging and Monitoring
```python
# shared_logging.py
import logging
import sys
import os

def setup_shared_logging(service_name):
    """Setup logging for multi-application scenarios"""
    log_format = f'[{service_name}] %(asctime)s - %(levelname)s - %(message)s'
    
    logging.basicConfig(
        level=logging.INFO,
        format=log_format,
        stream=sys.stdout
    )
    
    return logging.getLogger(service_name)
```

### 3. Configuration Management
```python
# config_manager.py
import os
import json

class ConfigManager:
    def __init__(self):
        self.config = self.load_config()
    
    def load_config(self):
        """Load configuration from environment and files"""
        config = {
            "service_mode": os.environ.get("SERVICE_MODE", "web"),
            "port": int(os.environ.get("PORT", 8080)),
            "debug": os.environ.get("DEBUG", "false").lower() == "true"
        }
        
        # Load from config file if exists
        config_file = os.environ.get("CONFIG_FILE", "/app/config.json")
        if os.path.exists(config_file):
            with open(config_file, 'r') as f:
                file_config = json.load(f)
                config.update(file_config)
        
        return config
    
    def get(self, key, default=None):
        return self.config.get(key, default)
```

### 4. Error Handling and Recovery
```python
# error_handler.py
import sys
import traceback
import signal

class ErrorHandler:
    def __init__(self, service_name):
        self.service_name = service_name
        self.setup_signal_handlers()
    
    def setup_signal_handlers(self):
        signal.signal(signal.SIGTERM, self.handle_shutdown)
        signal.signal(signal.SIGINT, self.handle_shutdown)
    
    def handle_shutdown(self, signum, frame):
        print(f"[{self.service_name}] Received shutdown signal")
        # Perform cleanup
        sys.exit(0)
    
    def handle_exception(self, exc_type, exc_value, exc_traceback):
        print(f"[{self.service_name}] Unhandled exception:")
        traceback.print_exception(exc_type, exc_value, exc_traceback)
        sys.exit(1)
```

### 5. Testing Multi-Application Scenarios
```bash
#!/bin/bash
# test_multi_app.sh

echo "Testing multi-application scenarios..."

# Test each configuration
for config in web worker admin java-web; do
    echo "Testing configuration: $config"
    
    # Start the application
    capstan run multi-app --boot $config &
    APP_PID=$!
    
    # Wait for startup
    sleep 5
    
    # Test if it's responding (if it's a web service)
    if [[ "$config" == *"web"* ]]; then
        curl -f http://localhost:8080/ || echo "Web test failed for $config"
    fi
    
    # Stop the application
    kill $APP_PID 2>/dev/null || true
    wait $APP_PID 2>/dev/null || true
    
    echo "Configuration $config test completed"
done

echo "All tests completed"
```

This comprehensive guide covers the various patterns and approaches for implementing multi-application scenarios in OSv using Capstan. The key is to understand OSv's single-process limitation and work within that constraint using appropriate architectural patterns.


