# Running Multiple Jobs and Multi-ELF Applications

This guide explains how to run multiple processes, jobs, or ELF executables within a single OSv unikernel using Capstan. This is useful for scenarios like running multiple services, background workers, or complex applications.

## Table of Contents

- [Understanding OSv Process Model](#understanding-osv-process-model)
- [Multi-Job Approaches](#multi-job-approaches)
- [Using Runscripts](#using-runscripts)
- [Python Multi-Job Examples](#python-multi-job-examples)
- [Java Multi-Job Examples](#java-multi-job-examples)
- [Native Multi-ELF Examples](#native-multi-elf-examples)
- [Advanced Patterns](#advanced-patterns)
- [Best Practices](#best-practices)

## Understanding OSv Process Model

### Important Limitations

OSv is a unikernel, which means:

1. **No fork()**: You cannot use `fork()` to create child processes
2. **Single process space**: All code runs in a single address space
3. **Multiple threads**: You CAN use threads for concurrency
4. **Multiple executables**: You CAN run multiple programs sequentially or in background threads

### What You CAN Do

- Run multiple applications **sequentially**
- Run background jobs using **threads**
- Execute multiple **ELF binaries** in sequence
- Use **runscripts** to orchestrate multiple commands
- Start **background threads** for concurrent execution

### What You CANNOT Do

- Use `fork()` or `execve()` system calls
- Run completely isolated processes
- Use traditional Unix process management (ps, kill by PID, etc.)

## Multi-Job Approaches

There are several ways to run multiple jobs in OSv:

### Approach 1: Sequential Execution

Run jobs one after another:

```bash
# In boot command
/job1.so && /job2.so && /job3.so
```

### Approach 2: Application-Level Threading

Use your application's threading to run multiple tasks:

**Python:**
```python
import threading

def worker1():
    # Job 1 code
    pass

def worker2():
    # Job 2 code
    pass

thread1 = threading.Thread(target=worker1)
thread2 = threading.Thread(target=worker2)
thread1.start()
thread2.start()
```

**Java:**
```java
Thread worker1 = new Thread(() -> {
    // Job 1 code
});
Thread worker2 = new Thread(() -> {
    // Job 2 code
});
worker1.start();
worker2.start();
```

### Approach 3: Runscripts

Use OSv runscripts to define complex execution flows with multiple commands.

## Using Runscripts

Runscripts are OSv's mechanism for running multiple commands, including background execution.

### Creating a Runscript

A runscript is a simple text file with commands:

```bash
# /init.yaml
# Format: each line is a command

# Start background services (& means background thread)
/server1.so &
/server2.so &

# Run foreground application
/main-app.so
```

### Runscript Syntax

**Sequential execution:**
```bash
/command1.so
/command2.so
/command3.so
```

**Background execution (concurrent):**
```bash
/background-service.so &
/main-app.so
```

**With arguments:**
```bash
/app.so --config /etc/config.yaml &
python3 /server.py --port 8000
```

**Environment variables:**
```bash
# Set environment in run.yaml, all commands inherit them
```

### Using Runscripts in Capstan

**Create runscript file:**

```bash
# Create init script
cat > init.yaml << 'EOF'
/usr/bin/service1 &
/usr/bin/service2 &
python3 /app.py
EOF
```

**Configure in run.yaml:**

```yaml
# meta/run.yaml
runtime: native

config_set:
  default:
    bootcmd: "runscript /init.yaml"

config_set_default: default
```

**Alternative: inline commands:**

```yaml
# meta/run.yaml
runtime: native

config_set:
  default:
    bootcmd: "/service1.so & /service2.so & /main.so"

config_set_default: default
```

## Python Multi-Job Examples

### Example 1: Web Server + Background Worker

Run a Flask web server and a background worker together:

**File: app.py**
```python
#!/usr/bin/env python3
from flask import Flask
import threading
import time

app = Flask(__name__)

# Background worker function
def background_worker():
    while True:
        print("Background worker running...")
        # Do background work
        time.sleep(10)

# Web server routes
@app.route('/')
def home():
    return "Web server running"

@app.route('/status')
def status():
    return {"status": "ok", "worker": "running"}

if __name__ == '__main__':
    # Start background worker in thread
    worker = threading.Thread(target=background_worker, daemon=True)
    worker.start()
    
    # Start web server (blocking)
    app.run(host='0.0.0.0', port=5000)
```

**Configuration:**
```yaml
# meta/package.yaml
name: com.example.multi-job-python
require:
  - osv.python3x
```

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

### Example 2: Multiple Python Scripts

Run multiple Python scripts using a wrapper:

**File: launcher.py**
```python
#!/usr/bin/env python3
import threading
import subprocess
import sys

def run_script(script_path):
    """Run a Python script"""
    print(f"Starting {script_path}")
    # In OSv, we can't use subprocess, so import and run directly
    import importlib.util
    spec = importlib.util.spec_from_file_location("module", script_path)
    module = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(module)

def run_in_thread(script_path):
    """Run script in a thread"""
    thread = threading.Thread(target=run_script, args=(script_path,))
    thread.daemon = False
    thread.start()
    return thread

if __name__ == '__main__':
    threads = []
    
    # Start multiple scripts
    threads.append(run_in_thread('/worker1.py'))
    threads.append(run_in_thread('/worker2.py'))
    threads.append(run_in_thread('/api_server.py'))
    
    # Wait for all to complete
    for t in threads:
        t.join()
```

### Example 3: Producer-Consumer Pattern

**File: app.py**
```python
#!/usr/bin/env python3
import threading
import queue
import time

# Shared queue
work_queue = queue.Queue()

def producer():
    """Producer thread"""
    counter = 0
    while True:
        item = f"item-{counter}"
        work_queue.put(item)
        print(f"Produced: {item}")
        counter += 1
        time.sleep(1)

def consumer(consumer_id):
    """Consumer thread"""
    while True:
        item = work_queue.get()
        print(f"Consumer {consumer_id} processing: {item}")
        time.sleep(2)  # Simulate work
        work_queue.task_done()

if __name__ == '__main__':
    # Start producer
    producer_thread = threading.Thread(target=producer, daemon=True)
    producer_thread.start()
    
    # Start multiple consumers
    for i in range(3):
        consumer_thread = threading.Thread(
            target=consumer, 
            args=(i,), 
            daemon=True
        )
        consumer_thread.start()
    
    # Keep main thread alive
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        print("Shutting down...")
```

### Example 4: Using Runscript for Multiple Python Apps

**File: worker1.py**
```python
#!/usr/bin/env python3
import time

while True:
    print("Worker 1 running")
    time.sleep(5)
```

**File: worker2.py**
```python
#!/usr/bin/env python3
import time

while True:
    print("Worker 2 running")
    time.sleep(5)
```

**File: server.py**
```python
#!/usr/bin/env python3
from flask import Flask

app = Flask(__name__)

@app.route('/')
def home():
    return "Server running with workers"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

**Runscript: /init.yaml**
```
python3 /worker1.py &
python3 /worker2.py &
python3 /server.py
```

**Configuration:**
```yaml
# meta/run.yaml
runtime: native

config_set:
  default:
    bootcmd: "runscript /init.yaml"
    env:
      PYTHONPATH: /packages

config_set_default: default
```

## Java Multi-Job Examples

### Example 1: Multiple Java Threads

**File: MultiJobApp.java**
```java
public class MultiJobApp {
    
    static class Worker implements Runnable {
        private String name;
        
        public Worker(String name) {
            this.name = name;
        }
        
        @Override
        public void run() {
            while (true) {
                System.out.println(name + " is working...");
                try {
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    break;
                }
            }
        }
    }
    
    static class WebServer implements Runnable {
        @Override
        public void run() {
            // Start your web server here
            System.out.println("Web server starting...");
            // Example: start Jetty, Tomcat, or custom server
        }
    }
    
    public static void main(String[] args) {
        // Start worker threads
        Thread worker1 = new Thread(new Worker("Worker-1"));
        Thread worker2 = new Thread(new Worker("Worker-2"));
        Thread server = new Thread(new WebServer());
        
        worker1.setDaemon(false);
        worker2.setDaemon(false);
        server.setDaemon(false);
        
        worker1.start();
        worker2.start();
        server.start();
        
        // Wait for threads
        try {
            worker1.join();
            worker2.join();
            server.join();
        } catch (InterruptedException e) {
            System.err.println("Interrupted");
        }
    }
}
```

**Configuration:**
```yaml
# meta/package.yaml
name: com.example.java-multijob
require:
  - osv.run-java
```

```yaml
# meta/run.yaml
runtime: java

config_set:
  default:
    main: MultiJobApp
    classpath:
      - /

config_set_default: default
```

### Example 2: Using ExecutorService

**File: MultiJobWithExecutor.java**
```java
import java.util.concurrent.*;

public class MultiJobWithExecutor {
    
    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(4);
        
        // Submit multiple tasks
        executor.submit(() -> {
            while (true) {
                System.out.println("Task 1 executing");
                Thread.sleep(5000);
            }
        });
        
        executor.submit(() -> {
            while (true) {
                System.out.println("Task 2 executing");
                Thread.sleep(5000);
            }
        });
        
        executor.submit(() -> {
            // Start web server
            startWebServer();
        });
        
        // Keep main thread alive
        try {
            Thread.sleep(Long.MAX_VALUE);
        } catch (InterruptedException e) {
            executor.shutdown();
        }
    }
    
    static void startWebServer() {
        // Your web server code
    }
}
```

## Native Multi-ELF Examples

### Example 1: Sequential ELF Execution

Run multiple native executables in sequence:

```yaml
# meta/run.yaml
runtime: native

config_set:
  default:
    bootcmd: "/setup.so && /process-data.so && /cleanup.so"

config_set_default: default
```

### Example 2: Background Services with Runscript

**File: /init.yaml**
```
# Start background services
/usr/sbin/redis-server /etc/redis.conf &
/usr/bin/nginx -c /etc/nginx.conf &

# Start main application
/app/main.so
```

**Configuration:**
```yaml
# meta/run.yaml
runtime: native

config_set:
  default:
    bootcmd: "runscript /init.yaml"

config_set_default: default
```

### Example 3: Multiple Services

**File: /services.yaml**
```
# Database
/usr/bin/postgres -D /var/lib/postgresql/data &

# Cache
/usr/bin/redis-server /etc/redis/redis.conf &

# Application
/app/server.so --port 8080
```

## Advanced Patterns

### Pattern 1: Service Orchestration

Create a master script that manages multiple services:

**Python example: orchestrator.py**
```python
#!/usr/bin/env python3
import threading
import time
import signal
import sys

services = []

class Service:
    def __init__(self, name, target):
        self.name = name
        self.target = target
        self.thread = None
        self.should_stop = False
    
    def start(self):
        self.thread = threading.Thread(target=self._run)
        self.thread.daemon = True
        self.thread.start()
        print(f"Started service: {self.name}")
    
    def _run(self):
        try:
            self.target()
        except Exception as e:
            print(f"Service {self.name} failed: {e}")
    
    def stop(self):
        self.should_stop = True
        print(f"Stopping service: {self.name}")

def service1():
    while True:
        print("Service 1 running")
        time.sleep(5)

def service2():
    while True:
        print("Service 2 running")
        time.sleep(5)

def main_app():
    from flask import Flask
    app = Flask(__name__)
    
    @app.route('/')
    def home():
        return "Main app with services"
    
    app.run(host='0.0.0.0', port=5000)

if __name__ == '__main__':
    # Register services
    services.append(Service("worker1", service1))
    services.append(Service("worker2", service2))
    
    # Start all services
    for service in services:
        service.start()
    
    # Run main app (blocking)
    try:
        main_app()
    except KeyboardInterrupt:
        print("Shutting down all services...")
        for service in services:
            service.stop()
```

### Pattern 2: Health Checking

Monitor and restart failed jobs:

```python
#!/usr/bin/env python3
import threading
import time

class MonitoredJob:
    def __init__(self, name, target):
        self.name = name
        self.target = target
        self.running = False
    
    def run(self):
        while True:
            try:
                print(f"Starting {self.name}")
                self.running = True
                self.target()
            except Exception as e:
                print(f"{self.name} crashed: {e}")
                self.running = False
                time.sleep(5)  # Wait before restart
                print(f"Restarting {self.name}")

def worker():
    # Your worker code
    while True:
        print("Working...")
        time.sleep(1)

if __name__ == '__main__':
    job = MonitoredJob("Worker", worker)
    thread = threading.Thread(target=job.run)
    thread.start()
    thread.join()
```

### Pattern 3: Job Coordination

Use queues to coordinate multiple jobs:

```python
#!/usr/bin/env python3
import threading
import queue
import time

# Shared state
task_queue = queue.Queue()
result_queue = queue.Queue()

def producer():
    for i in range(100):
        task_queue.put(f"task-{i}")
        time.sleep(0.1)
    task_queue.put(None)  # Sentinel

def worker(worker_id):
    while True:
        task = task_queue.get()
        if task is None:
            task_queue.put(None)  # For other workers
            break
        
        # Process task
        result = f"Worker-{worker_id} processed {task}"
        result_queue.put(result)

def collector():
    while True:
        result = result_queue.get()
        if result is None:
            break
        print(result)

if __name__ == '__main__':
    # Start workers
    workers = []
    for i in range(4):
        t = threading.Thread(target=worker, args=(i,))
        t.start()
        workers.append(t)
    
    # Start producer
    producer_thread = threading.Thread(target=producer)
    producer_thread.start()
    
    # Start collector
    collector_thread = threading.Thread(target=collector)
    collector_thread.start()
    
    # Wait for completion
    producer_thread.join()
    for w in workers:
        w.join()
    result_queue.put(None)
    collector_thread.join()
```

## Best Practices

### 1. Use Threading, Not Processes

❌ **Don't use:**
```python
import subprocess
subprocess.Popen(['python', 'worker.py'])  # Won't work in OSv
```

✅ **Do use:**
```python
import threading
thread = threading.Thread(target=worker_function)
thread.start()
```

### 2. Handle Graceful Shutdown

```python
import signal
import sys

def signal_handler(sig, frame):
    print('Shutting down gracefully...')
    # Cleanup code here
    sys.exit(0)

signal.signal(signal.SIGINT, signal_handler)
signal.signal(signal.SIGTERM, signal_handler)
```

### 3. Use Daemon Threads Carefully

```python
# Daemon threads die with main thread
worker = threading.Thread(target=func, daemon=True)

# Non-daemon keeps process alive
worker = threading.Thread(target=func, daemon=False)
```

### 4. Log Everything

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(message)s'
)

logger = logging.getLogger(__name__)
logger.info("Service started")
```

### 5. Use Thread-Safe Data Structures

```python
import queue
import threading

# Thread-safe queue
q = queue.Queue()

# Thread-safe operations
lock = threading.Lock()
with lock:
    # Critical section
    pass
```

## Troubleshooting

### Jobs Not Starting

**Problem**: Background jobs don't start

**Solution**: Ensure main thread doesn't exit:
```python
# Keep main thread alive
import time
while True:
    time.sleep(1)
```

### Jobs Exit Prematurely

**Problem**: Jobs exit unexpectedly

**Solution**: Use non-daemon threads or keep main thread running:
```python
thread.daemon = False  # Thread won't die with main
thread.start()
thread.join()  # Wait for thread
```

### Cannot Fork Error

**Problem**: `OSError: [Errno 38] Function not implemented`

**Solution**: Don't use fork, use threading:
```python
# Wrong
os.fork()

# Right
threading.Thread(target=func).start()
```

### High CPU Usage

**Problem**: Multiple jobs consume too much CPU

**Solution**: Add sleep/yield in loops:
```python
while True:
    do_work()
    time.sleep(0.01)  # Yield CPU
```

## Summary

| Approach | Use Case | Example |
|----------|----------|---------|
| Threading | Python/Java concurrent tasks | `threading.Thread()` |
| Runscripts | Multiple native binaries | `runscript /init.yaml` |
| Sequential | Ordered execution | `/cmd1 && /cmd2` |
| Background | Long-running services | `/service &` in runscript |
| Orchestration | Complex multi-service apps | Custom manager script |

## Next Steps

- Review [Python Workflow](PythonWorkflow.md) for Python-specific patterns
- See [Configuration Files](ConfigurationFiles.md) for runtime options
- Explore [RuntimeNative](RuntimeNative.md) for native ELF execution
- Check [RuntimeJava](RuntimeJava.md) for Java threading examples
