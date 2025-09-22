# Chapter 6: Process Management and Child Processes

At some point in every CLI tool's life, it realizes it can't do everything alone. Maybe it needs to run git commands, or compile TypeScript, or spin up a development server. This is when your CLI tool becomes a parent—spawning child processes, managing them, and occasionally wondering where it all went wrong when one of them refuses to terminate.

Process management is what separates toy CLI tools from production ones. It's the difference between a script that runs a command and a tool that orchestrates complex workflows. It's also where you learn that computers are basically just processes all the way down, like some kind of computational nesting doll.

## The Process Trinity: spawn, exec, and fork

Node.js gives you three main ways to create child processes, each with its own personality quirks:

```javascript
import { spawn, exec, fork } from 'child_process';
import { promisify } from 'util';
const execPromise = promisify(exec);

// spawn: The streaming powerhouse
// Use when: You need streaming I/O, long-running processes, or fine control
const ls = spawn('ls', ['-la', '/usr']);
ls.stdout.on('data', (data) => {
  console.log(`stdout: ${data}`);
});
ls.on('close', (code) => {
  console.log(`child process exited with code ${code}`);
});

// exec: The simple solution
// Use when: You need the output as a string and the command is simple
exec('git status', (error, stdout, stderr) => {
  if (error) {
    console.error(`exec error: ${error}`);
    return;
  }
  console.log(`stdout: ${stdout}`);
});

// fork: The Node.js special
// Use when: You need another Node.js process with IPC
const worker = fork('worker.js');
worker.send({ cmd: 'start', data: 'payload' });
worker.on('message', (msg) => {
  console.log('Message from worker:', msg);
});
```

But here's the thing: knowing which one to use is an art form. Let's explore each in detail.

## spawn: The Swiss Army Knife

`spawn` is the foundation of all child process methods. It's powerful, flexible, and occasionally frustrating.

```javascript
import { spawn } from 'child_process';
import chalk from 'chalk';

class ProcessManager {
  constructor() {
    this.processes = new Map();
  }

  async run(command, args = [], options = {}) {
    return new Promise((resolve, reject) => {
      const defaultOptions = {
        stdio: 'pipe',
        shell: false,
        ...options
      };

      console.log(chalk.gray(`> ${command} ${args.join(' ')}`));

      const child = spawn(command, args, defaultOptions);
      const processId = Date.now();

      this.processes.set(processId, {
        process: child,
        command: `${command} ${args.join(' ')}`,
        startTime: Date.now()
      });

      let stdout = '';
      let stderr = '';

      if (child.stdout) {
        child.stdout.on('data', (data) => {
          stdout += data.toString();
          if (options.onStdout) {
            options.onStdout(data.toString());
          }
        });
      }

      if (child.stderr) {
        child.stderr.on('data', (data) => {
          stderr += data.toString();
          if (options.onStderr) {
            options.onStderr(data.toString());
          }
        });
      }

      child.on('error', (error) => {
        this.processes.delete(processId);
        reject(error);
      });

      child.on('close', (code, signal) => {
        this.processes.delete(processId);

        const duration = Date.now() - this.processes.get(processId).startTime;

        if (code === 0) {
          console.log(chalk.green(`✓ Completed in ${duration}ms`));
          resolve({ stdout, stderr, code, signal, duration });
        } else {
          const error = new Error(`Process exited with code ${code}`);
          error.stdout = stdout;
          error.stderr = stderr;
          error.code = code;
          error.signal = signal;
          reject(error);
        }
      });
    });
  }

  async runWithTimeout(command, args, timeout = 5000) {
    return Promise.race([
      this.run(command, args),
      new Promise((_, reject) =>
        setTimeout(() => reject(new Error(`Timeout after ${timeout}ms`)), timeout)
      )
    ]);
  }

  killAll() {
    for (const [id, info] of this.processes) {
      console.log(chalk.yellow(`Terminating: ${info.command}`));
      info.process.kill('SIGTERM');
    }
  }
}
```

## Streaming Output: Real-time Feedback

Users want to see what's happening. Streaming output gives them that visibility:

```javascript
class StreamingExecutor {
  async streamCommand(command, args, options = {}) {
    const {
      prefix = '',
      color = 'white',
      silent = false
    } = options;

    const child = spawn(command, args, {
      stdio: ['inherit', 'pipe', 'pipe'],
      ...options
    });

    // Create transform streams for formatting
    const stdoutTransform = new Transform({
      transform(chunk, encoding, callback) {
        const lines = chunk.toString().split('\n');
        const formatted = lines
          .filter(line => line.trim())
          .map(line => `${chalk[color](prefix)}${line}`)
          .join('\n');

        if (!silent && formatted) {
          console.log(formatted);
        }

        callback(null, chunk);
      }
    });

    const stderrTransform = new Transform({
      transform(chunk, encoding, callback) {
        const lines = chunk.toString().split('\n');
        const formatted = lines
          .filter(line => line.trim())
          .map(line => `${chalk.red(prefix)}${line}`)
          .join('\n');

        if (!silent && formatted) {
          console.error(formatted);
        }

        callback(null, chunk);
      }
    });

    child.stdout.pipe(stdoutTransform);
    child.stderr.pipe(stderrTransform);

    return new Promise((resolve, reject) => {
      child.on('close', (code) => {
        if (code === 0) {
          resolve(code);
        } else {
          reject(new Error(`Process exited with code ${code}`));
        }
      });

      child.on('error', reject);
    });
  }

  async runParallel(commands) {
    const colors = ['cyan', 'magenta', 'yellow', 'green', 'blue'];

    const promises = commands.map((cmd, index) => {
      const [command, ...args] = cmd.split(' ');
      const color = colors[index % colors.length];
      const prefix = `[${command}] `;

      return this.streamCommand(command, args, { prefix, color });
    });

    return Promise.all(promises);
  }
}

// Usage
const executor = new StreamingExecutor();
await executor.runParallel([
  'npm run build',
  'npm run test',
  'npm run lint'
]);
```

## exec: When You Just Want It Done

Sometimes you don't need streams or fine control. You just want to run a command and get the output:

```javascript
import { exec } from 'child_process';
import { promisify } from 'util';
const execAsync = promisify(exec);

class ShellExecutor {
  constructor() {
    this.history = [];
    this.env = { ...process.env };
  }

  async exec(command, options = {}) {
    const startTime = Date.now();

    const execOptions = {
      encoding: 'utf8',
      maxBuffer: 1024 * 1024 * 10, // 10MB
      env: this.env,
      ...options
    };

    try {
      const { stdout, stderr } = await execAsync(command, execOptions);

      const result = {
        command,
        stdout: stdout.trim(),
        stderr: stderr.trim(),
        duration: Date.now() - startTime,
        timestamp: new Date().toISOString(),
        success: true
      };

      this.history.push(result);
      return result;
    } catch (error) {
      const result = {
        command,
        stdout: error.stdout?.trim() || '',
        stderr: error.stderr?.trim() || error.message,
        code: error.code,
        duration: Date.now() - startTime,
        timestamp: new Date().toISOString(),
        success: false,
        error: error.message
      };

      this.history.push(result);
      throw error;
    }
  }

  async execSilent(command) {
    try {
      const result = await this.exec(command);
      return result.stdout;
    } catch (error) {
      return null;
    }
  }

  async pipe(commands) {
    const joined = commands.join(' | ');
    return this.exec(joined, { shell: true });
  }

  async getGitInfo() {
    const [branch, status, lastCommit] = await Promise.all([
      this.execSilent('git branch --show-current'),
      this.execSilent('git status --porcelain'),
      this.execSilent('git log -1 --oneline')
    ]);

    return {
      branch,
      isDirty: status && status.length > 0,
      lastCommit
    };
  }
}
```

## fork: The Node.js Multiverse

When you need to run Node.js code in a separate process with built-in communication:

```javascript
// main.js
import { fork } from 'child_process';
import { fileURLToPath } from 'url';
import { dirname, join } from 'path';

const __dirname = dirname(fileURLToPath(import.meta.url));

class WorkerPool {
  constructor(workerScript, poolSize = 4) {
    this.workerScript = workerScript;
    this.poolSize = poolSize;
    this.workers = [];
    this.queue = [];
    this.activeJobs = new Map();

    this.initializePool();
  }

  initializePool() {
    for (let i = 0; i < this.poolSize; i++) {
      this.createWorker();
    }
  }

  createWorker() {
    const worker = fork(this.workerScript, [], {
      silent: false,
      execArgv: process.execArgv
    });

    worker.on('message', (msg) => {
      if (msg.type === 'result') {
        const job = this.activeJobs.get(msg.id);
        if (job) {
          job.resolve(msg.data);
          this.activeJobs.delete(msg.id);
          this.processQueue(worker);
        }
      } else if (msg.type === 'error') {
        const job = this.activeJobs.get(msg.id);
        if (job) {
          job.reject(new Error(msg.error));
          this.activeJobs.delete(msg.id);
          this.processQueue(worker);
        }
      }
    });

    worker.on('error', (error) => {
      console.error('Worker error:', error);
      this.handleWorkerError(worker);
    });

    worker.on('exit', (code, signal) => {
      if (code !== 0) {
        console.error(`Worker exited with code ${code}`);
        this.handleWorkerExit(worker);
      }
    });

    this.workers.push(worker);
    return worker;
  }

  async execute(task) {
    return new Promise((resolve, reject) => {
      const jobId = Date.now() + Math.random();
      const job = { id: jobId, task, resolve, reject };

      const availableWorker = this.workers.find(w =>
        !Array.from(this.activeJobs.values()).some(j => j.worker === w)
      );

      if (availableWorker) {
        this.sendToWorker(availableWorker, job);
      } else {
        this.queue.push(job);
      }
    });
  }

  sendToWorker(worker, job) {
    this.activeJobs.set(job.id, { ...job, worker });
    worker.send({ id: job.id, type: 'task', data: job.task });
  }

  processQueue(worker) {
    if (this.queue.length > 0) {
      const job = this.queue.shift();
      this.sendToWorker(worker, job);
    }
  }

  handleWorkerError(worker) {
    const index = this.workers.indexOf(worker);
    if (index !== -1) {
      this.workers.splice(index, 1);
      this.createWorker();
    }
  }

  handleWorkerExit(worker) {
    this.handleWorkerError(worker);
  }

  async shutdown() {
    for (const worker of this.workers) {
      worker.kill('SIGTERM');
    }
    this.workers = [];
    this.queue = [];
    this.activeJobs.clear();
  }
}

// worker.js
process.on('message', async (msg) => {
  if (msg.type === 'task') {
    try {
      // Simulate heavy computation
      const result = await performHeavyTask(msg.data);
      process.send({ type: 'result', id: msg.id, data: result });
    } catch (error) {
      process.send({ type: 'error', id: msg.id, error: error.message });
    }
  }
});

async function performHeavyTask(data) {
  // Simulate CPU-intensive work
  const start = Date.now();
  let result = 0;
  for (let i = 0; i < 1000000; i++) {
    result += Math.sqrt(i);
  }
  return {
    input: data,
    result,
    duration: Date.now() - start
  };
}
```

## Process Communication: The Art of IPC

Inter-Process Communication (IPC) is how processes talk to each other. It's like texting, but for programs:

```javascript
class IPCManager {
  constructor() {
    this.channels = new Map();
    this.handlers = new Map();
  }

  createChannel(name, processPath) {
    const channel = fork(processPath, [], {
      stdio: ['pipe', 'pipe', 'pipe', 'ipc']
    });

    channel.on('message', (msg) => {
      this.handleMessage(name, msg);
    });

    this.channels.set(name, channel);
    return channel;
  }

  on(channel, event, handler) {
    const key = `${channel}:${event}`;
    if (!this.handlers.has(key)) {
      this.handlers.set(key, []);
    }
    this.handlers.get(key).push(handler);
  }

  emit(channel, event, data) {
    const ch = this.channels.get(channel);
    if (ch) {
      ch.send({ event, data });
    }
  }

  handleMessage(channel, msg) {
    const key = `${channel}:${msg.event}`;
    const handlers = this.handlers.get(key) || [];
    handlers.forEach(handler => handler(msg.data));
  }

  async request(channel, event, data, timeout = 5000) {
    return new Promise((resolve, reject) => {
      const ch = this.channels.get(channel);
      if (!ch) {
        reject(new Error(`Channel ${channel} not found`));
        return;
      }

      const requestId = Date.now() + Math.random();
      const timer = setTimeout(() => {
        reject(new Error(`Request timeout after ${timeout}ms`));
      }, timeout);

      const responseHandler = (msg) => {
        if (msg.requestId === requestId) {
          clearTimeout(timer);
          ch.removeListener('message', responseHandler);
          resolve(msg.data);
        }
      };

      ch.on('message', responseHandler);
      ch.send({ event, data, requestId });
    });
  }

  broadcast(event, data) {
    for (const [name, channel] of this.channels) {
      channel.send({ event, data });
    }
  }

  close(channel) {
    const ch = this.channels.get(channel);
    if (ch) {
      ch.kill('SIGTERM');
      this.channels.delete(channel);
    }
  }

  closeAll() {
    for (const [name, channel] of this.channels) {
      channel.kill('SIGTERM');
    }
    this.channels.clear();
  }
}
```

## Process Orchestration: Conducting the Symphony

Real CLI tools don't just run processes; they orchestrate them:

```javascript
class ProcessOrchestrator {
  constructor() {
    this.tasks = new Map();
    this.dependencies = new Map();
    this.running = new Set();
    this.completed = new Set();
    this.failed = new Set();
  }

  addTask(name, command, options = {}) {
    this.tasks.set(name, {
      name,
      command,
      dependencies: options.dependencies || [],
      retries: options.retries || 0,
      timeout: options.timeout || null,
      parallel: options.parallel || false
    });

    // Build dependency graph
    if (options.dependencies) {
      for (const dep of options.dependencies) {
        if (!this.dependencies.has(dep)) {
          this.dependencies.set(dep, new Set());
        }
        this.dependencies.get(dep).add(name);
      }
    }
  }

  async run() {
    console.log(chalk.bold('\n🎭 Process Orchestrator\n'));

    // Find tasks with no dependencies
    const initialTasks = [];
    for (const [name, task] of this.tasks) {
      if (task.dependencies.length === 0) {
        initialTasks.push(name);
      }
    }

    // Start execution
    await this.executeTasks(initialTasks);

    // Check if all tasks completed
    if (this.completed.size === this.tasks.size) {
      console.log(chalk.green('\n✓ All tasks completed successfully!'));
    } else {
      console.log(chalk.red(`\n✗ ${this.failed.size} tasks failed`));
      for (const task of this.failed) {
        console.log(chalk.red(`  - ${task}`));
      }
    }
  }

  async executeTasks(taskNames) {
    const promises = [];

    for (const name of taskNames) {
      if (this.running.has(name) || this.completed.has(name)) {
        continue;
      }

      const task = this.tasks.get(name);
      if (!task) continue;

      // Check if dependencies are satisfied
      const ready = task.dependencies.every(dep =>
        this.completed.has(dep)
      );

      if (!ready) continue;

      this.running.add(name);
      promises.push(this.executeTask(task));
    }

    if (promises.length > 0) {
      await Promise.all(promises);
    }
  }

  async executeTask(task) {
    const spinner = ora(`Running: ${task.name}`).start();

    try {
      const [command, ...args] = task.command.split(' ');

      await this.runWithRetry(command, args, task);

      spinner.succeed(`Completed: ${task.name}`);
      this.completed.add(task.name);
      this.running.delete(task.name);

      // Check for dependent tasks
      const dependents = this.dependencies.get(task.name) || new Set();
      if (dependents.size > 0) {
        await this.executeTasks(Array.from(dependents));
      }
    } catch (error) {
      spinner.fail(`Failed: ${task.name}`);
      this.failed.add(task.name);
      this.running.delete(task.name);

      // Cancel dependent tasks
      this.cancelDependents(task.name);
    }
  }

  async runWithRetry(command, args, task) {
    let lastError;

    for (let i = 0; i <= task.retries; i++) {
      try {
        if (i > 0) {
          console.log(chalk.yellow(`  Retry ${i}/${task.retries}`));
        }

        if (task.timeout) {
          await this.runWithTimeout(command, args, task.timeout);
        } else {
          await this.runCommand(command, args);
        }

        return;
      } catch (error) {
        lastError = error;
        if (i < task.retries) {
          await this.delay(1000 * Math.pow(2, i)); // Exponential backoff
        }
      }
    }

    throw lastError;
  }

  async runWithTimeout(command, args, timeout) {
    return Promise.race([
      this.runCommand(command, args),
      new Promise((_, reject) =>
        setTimeout(() => reject(new Error('Timeout')), timeout)
      )
    ]);
  }

  async runCommand(command, args) {
    return new Promise((resolve, reject) => {
      const child = spawn(command, args, {
        stdio: 'pipe'
      });

      let stderr = '';

      child.stderr.on('data', (data) => {
        stderr += data.toString();
      });

      child.on('close', (code) => {
        if (code === 0) {
          resolve();
        } else {
          reject(new Error(`Exit code ${code}: ${stderr}`));
        }
      });

      child.on('error', reject);
    });
  }

  cancelDependents(taskName) {
    const dependents = this.dependencies.get(taskName) || new Set();
    for (const dependent of dependents) {
      if (!this.completed.has(dependent) && !this.failed.has(dependent)) {
        console.log(chalk.yellow(`  Skipped: ${dependent} (dependency failed)`));
        this.failed.add(dependent);
        this.cancelDependents(dependent);
      }
    }
  }

  delay(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// Usage
const orchestrator = new ProcessOrchestrator();

orchestrator.addTask('install', 'npm install');
orchestrator.addTask('build:css', 'npm run build:css', {
  dependencies: ['install']
});
orchestrator.addTask('build:js', 'npm run build:js', {
  dependencies: ['install']
});
orchestrator.addTask('build', 'npm run build:final', {
  dependencies: ['build:css', 'build:js']
});
orchestrator.addTask('test', 'npm test', {
  dependencies: ['build'],
  retries: 2,
  timeout: 30000
});
orchestrator.addTask('deploy', 'npm run deploy', {
  dependencies: ['test']
});

await orchestrator.run();
```

## Signal Handling: Graceful Shutdown

Processes need to die with dignity:

```javascript
class GracefulShutdown {
  constructor() {
    this.handlers = [];
    this.isShuttingDown = false;
    this.forceTimeout = 10000; // Force kill after 10 seconds

    this.setupSignalHandlers();
  }

  setupSignalHandlers() {
    const signals = ['SIGINT', 'SIGTERM', 'SIGQUIT'];

    signals.forEach(signal => {
      process.on(signal, async () => {
        if (this.isShuttingDown) {
          console.log(chalk.yellow('\n⚠ Force shutdown requested'));
          process.exit(1);
        }

        this.isShuttingDown = true;
        console.log(chalk.yellow(`\n📍 Received ${signal}, shutting down gracefully...`));

        await this.shutdown();
      });
    });

    // Windows-specific handling
    if (process.platform === 'win32') {
      const rl = require('readline').createInterface({
        input: process.stdin,
        output: process.stdout
      });

      rl.on('SIGINT', () => {
        process.emit('SIGINT');
      });
    }
  }

  onShutdown(handler) {
    this.handlers.push(handler);
  }

  async shutdown() {
    const timeout = setTimeout(() => {
      console.error(chalk.red('⚠ Shutdown timeout, forcing exit'));
      process.exit(1);
    }, this.forceTimeout);

    try {
      // Run all cleanup handlers in parallel
      await Promise.all(
        this.handlers.map(async (handler) => {
          try {
            await handler();
          } catch (error) {
            console.error(chalk.red(`Shutdown handler error: ${error.message}`));
          }
        })
      );

      clearTimeout(timeout);
      console.log(chalk.green('✓ Graceful shutdown complete'));
      process.exit(0);
    } catch (error) {
      clearTimeout(timeout);
      console.error(chalk.red(`Shutdown error: ${error.message}`));
      process.exit(1);
    }
  }
}

// Usage with process management
class ManagedProcessRunner {
  constructor() {
    this.processes = new Map();
    this.shutdown = new GracefulShutdown();

    this.shutdown.onShutdown(async () => {
      console.log('Terminating child processes...');
      await this.terminateAll();
    });
  }

  spawn(name, command, args, options = {}) {
    const child = spawn(command, args, options);

    this.processes.set(name, {
      process: child,
      command: `${command} ${args.join(' ')}`,
      startTime: Date.now()
    });

    child.on('exit', () => {
      this.processes.delete(name);
    });

    return child;
  }

  async terminateAll() {
    const promises = [];

    for (const [name, info] of this.processes) {
      promises.push(this.terminate(name));
    }

    await Promise.all(promises);
  }

  async terminate(name, timeout = 5000) {
    const info = this.processes.get(name);
    if (!info) return;

    return new Promise((resolve) => {
      const { process: child } = info;

      // Set up timeout
      const timer = setTimeout(() => {
        console.log(chalk.yellow(`Force killing ${name}`));
        child.kill('SIGKILL');
        resolve();
      }, timeout);

      // Listen for exit
      child.once('exit', () => {
        clearTimeout(timer);
        this.processes.delete(name);
        resolve();
      });

      // Try graceful termination first
      child.kill('SIGTERM');
    });
  }
}
```

## Process Monitoring: Keeping Watch

Sometimes you need to monitor long-running processes:

```javascript
class ProcessMonitor {
  constructor() {
    this.metrics = new Map();
    this.alerts = [];
  }

  monitor(child, name, options = {}) {
    const {
      memoryThreshold = 500 * 1024 * 1024, // 500MB
      cpuThreshold = 80, // 80%
      restartOnCrash = false,
      maxRestarts = 3
    } = options;

    const metrics = {
      name,
      pid: child.pid,
      startTime: Date.now(),
      restarts: 0,
      crashes: 0,
      memoryPeak: 0,
      cpuPeak: 0
    };

    this.metrics.set(child.pid, metrics);

    // Monitor resource usage
    const monitorInterval = setInterval(() => {
      try {
        const usage = process.memoryUsage();

        if (usage.rss > metrics.memoryPeak) {
          metrics.memoryPeak = usage.rss;
        }

        if (usage.rss > memoryThreshold) {
          this.alert(`High memory usage for ${name}: ${this.formatBytes(usage.rss)}`);
        }
      } catch (error) {
        clearInterval(monitorInterval);
      }
    }, 1000);

    // Monitor exit
    child.on('exit', (code, signal) => {
      clearInterval(monitorInterval);

      const runtime = Date.now() - metrics.startTime;
      console.log(chalk.gray(`Process ${name} exited after ${this.formatDuration(runtime)}`));

      if (code !== 0 && restartOnCrash && metrics.restarts < maxRestarts) {
        metrics.crashes++;
        metrics.restarts++;
        console.log(chalk.yellow(`Restarting ${name} (attempt ${metrics.restarts}/${maxRestarts})`));

        // Restart logic here
        setTimeout(() => {
          // Re-spawn the process
        }, 1000 * metrics.restarts); // Exponential backoff
      }

      this.metrics.delete(child.pid);
    });

    return metrics;
  }

  alert(message) {
    const alert = {
      message,
      timestamp: new Date().toISOString()
    };

    this.alerts.push(alert);
    console.log(chalk.red(`⚠ Alert: ${message}`));
  }

  getStatus() {
    const status = [];

    for (const [pid, metrics] of this.metrics) {
      const runtime = Date.now() - metrics.startTime;

      status.push({
        name: metrics.name,
        pid,
        runtime: this.formatDuration(runtime),
        memory: this.formatBytes(metrics.memoryPeak),
        restarts: metrics.restarts,
        crashes: metrics.crashes
      });
    }

    return status;
  }

  formatBytes(bytes) {
    const units = ['B', 'KB', 'MB', 'GB'];
    let size = bytes;
    let unit = 0;

    while (size > 1024 && unit < units.length - 1) {
      size /= 1024;
      unit++;
    }

    return `${size.toFixed(1)}${units[unit]}`;
  }

  formatDuration(ms) {
    if (ms < 1000) return `${ms}ms`;
    if (ms < 60000) return `${(ms / 1000).toFixed(1)}s`;
    if (ms < 3600000) return `${Math.floor(ms / 60000)}m ${Math.floor((ms % 60000) / 1000)}s`;
    return `${Math.floor(ms / 3600000)}h ${Math.floor((ms % 3600000) / 60000)}m`;
  }
}
```

## Advanced Patterns: Process Pools and Queues

For heavy workloads, you need sophisticated process management:

```javascript
class ProcessQueue {
  constructor(options = {}) {
    this.concurrency = options.concurrency || 4;
    this.timeout = options.timeout || 30000;
    this.retries = options.retries || 3;

    this.queue = [];
    this.running = new Map();
    this.results = new Map();
    this.errors = new Map();
  }

  enqueue(task) {
    return new Promise((resolve, reject) => {
      this.queue.push({
        id: Date.now() + Math.random(),
        task,
        resolve,
        reject,
        attempts: 0
      });

      this.processNext();
    });
  }

  async processNext() {
    if (this.running.size >= this.concurrency || this.queue.length === 0) {
      return;
    }

    const job = this.queue.shift();
    this.running.set(job.id, job);

    try {
      const result = await this.executeJob(job);
      this.results.set(job.id, result);
      job.resolve(result);
    } catch (error) {
      job.attempts++;

      if (job.attempts < this.retries) {
        // Retry with exponential backoff
        setTimeout(() => {
          this.queue.push(job);
          this.processNext();
        }, Math.pow(2, job.attempts) * 1000);
      } else {
        this.errors.set(job.id, error);
        job.reject(error);
      }
    } finally {
      this.running.delete(job.id);
      this.processNext();
    }
  }

  async executeJob(job) {
    const [command, ...args] = job.task.command.split(' ');

    return new Promise((resolve, reject) => {
      const child = spawn(command, args, {
        stdio: 'pipe'
      });

      let stdout = '';
      let stderr = '';
      let killed = false;

      const timeout = setTimeout(() => {
        killed = true;
        child.kill('SIGTERM');
        reject(new Error(`Job ${job.id} timed out`));
      }, this.timeout);

      child.stdout.on('data', (data) => {
        stdout += data.toString();
      });

      child.stderr.on('data', (data) => {
        stderr += data.toString();
      });

      child.on('close', (code) => {
        clearTimeout(timeout);

        if (killed) return;

        if (code === 0) {
          resolve({ stdout, stderr, code });
        } else {
          reject(new Error(`Process exited with code ${code}: ${stderr}`));
        }
      });

      child.on('error', (error) => {
        clearTimeout(timeout);
        reject(error);
      });
    });
  }

  async drain() {
    while (this.queue.length > 0 || this.running.size > 0) {
      await new Promise(resolve => setTimeout(resolve, 100));
    }
  }

  getStats() {
    return {
      queued: this.queue.length,
      running: this.running.size,
      completed: this.results.size,
      failed: this.errors.size
    };
  }
}
```

## Real-World Example: Build System

Let's put it all together with a build system:

```javascript
class BuildSystem {
  constructor() {
    this.monitor = new ProcessMonitor();
    this.queue = new ProcessQueue({ concurrency: 4 });
    this.orchestrator = new ProcessOrchestrator();
  }

  async build(config) {
    console.log(chalk.bold('\n🏗️  Build System v1.0.0\n'));

    // Parse build configuration
    const tasks = this.parseBuildConfig(config);

    // Setup build tasks
    for (const task of tasks) {
      this.orchestrator.addTask(task.name, task.command, {
        dependencies: task.dependencies,
        retries: task.retries || 0
      });
    }

    // Watch mode
    if (config.watch) {
      this.setupWatchers(config);
    }

    // Run build
    const startTime = Date.now();

    try {
      await this.orchestrator.run();

      const duration = Date.now() - startTime;
      console.log(chalk.green(`\n✓ Build completed in ${duration}ms`));

      // Run post-build tasks
      if (config.postBuild) {
        await this.runPostBuild(config.postBuild);
      }
    } catch (error) {
      console.error(chalk.red(`\n✗ Build failed: ${error.message}`));
      process.exit(1);
    }
  }

  parseBuildConfig(config) {
    const tasks = [];

    // TypeScript compilation
    if (config.typescript) {
      tasks.push({
        name: 'typescript',
        command: 'tsc',
        dependencies: []
      });
    }

    // Bundling
    if (config.bundle) {
      tasks.push({
        name: 'bundle',
        command: `webpack --mode ${config.mode || 'production'}`,
        dependencies: config.typescript ? ['typescript'] : []
      });
    }

    // Tests
    if (config.test) {
      tasks.push({
        name: 'test',
        command: 'npm test',
        dependencies: ['bundle'],
        retries: 2
      });
    }

    // Linting
    if (config.lint) {
      tasks.push({
        name: 'lint',
        command: 'eslint src',
        dependencies: []
      });
    }

    return tasks;
  }

  setupWatchers(config) {
    const watcher = chokidar.watch(config.watchPaths || ['src/**/*'], {
      ignored: /(^|[\/\\])\../,
      persistent: true
    });

    watcher.on('change', async (path) => {
      console.log(chalk.yellow(`\n📝 File changed: ${path}`));
      console.log(chalk.gray('Rebuilding...'));

      // Queue rebuild
      this.queue.enqueue({
        command: 'npm run build:incremental'
      });
    });
  }

  async runPostBuild(tasks) {
    console.log(chalk.bold('\n📦 Running post-build tasks...\n'));

    for (const task of tasks) {
      const spinner = ora(task.name).start();

      try {
        await this.queue.enqueue({ command: task.command });
        spinner.succeed();
      } catch (error) {
        spinner.fail();
        throw error;
      }
    }
  }
}

// Usage
const builder = new BuildSystem();
await builder.build({
  typescript: true,
  bundle: true,
  test: true,
  lint: true,
  watch: true,
  watchPaths: ['src/**/*.ts'],
  postBuild: [
    { name: 'Copy assets', command: 'cp -r assets dist/' },
    { name: 'Generate docs', command: 'npm run docs' },
    { name: 'Optimize images', command: 'npm run optimize:images' }
  ]
});
```

## Process Management Best Practices

After years of managing processes, here are the lessons learned:

1. **Always handle signals**: Processes die. Handle it gracefully.

2. **Set timeouts**: Hanging processes are worse than failed ones.

3. **Use the right tool**: `spawn` for streams, `exec` for simple commands, `fork` for Node.js IPC.

4. **Monitor resources**: Runaway processes eat memory. Watch them.

5. **Log everything**: When processes fail in production, logs are your only friend.

6. **Implement retry logic**: Transient failures happen. Retry with exponential backoff.

7. **Clean up zombies**: Dead processes that won't die are zombies. Kill them properly.

8. **Respect the platform**: Windows doesn't have signals like Unix. Handle both.

9. **Queue heavy work**: Don't spawn 1000 processes at once. Your computer will hate you.

10. **Test failure modes**: Kill your processes. See what breaks. Fix it.

## The Process Philosophy

Process management in Node.js is like parenting—you create these processes, nurture them, watch them grow, and eventually, you have to let them go (usually with SIGTERM). Some processes are well-behaved and exit cleanly. Others throw tantrums and need to be forcefully terminated. And occasionally, you get zombie processes that refuse to die properly, haunting your system like digital ghosts.

The key to good process management isn't just knowing the APIs—it's understanding the lifecycle, respecting the resources, and always, always having a plan for when things go wrong. Because in the world of processes, things always go wrong. It's not pessimism; it's experience.

In the next chapter, we'll explore how modern CLI tools are integrating AI, turning simple command-line interfaces into intelligent assistants. We'll see how process management becomes even more critical when you're orchestrating AI operations, streaming responses, and managing the complex dance between local processes and cloud APIs. The future of CLI tools is here, and it speaks GPT.