# Chapter 8: Performance and Optimization

"My CLI tool is too fast," said no developer ever. Performance in CLI tools is like money in your bank account—you can never have too much. Users notice when your tool takes 3 seconds to start. They notice when file operations drag. They especially notice when your "quick check" command takes longer than manually doing the task would have taken.

The paradox of Node.js performance is that it's both surprisingly fast and frustratingly slow. It's fast at I/O operations, which is great for CLI tools. It's slow at CPU-intensive tasks, which is... less great. It starts quickly compared to Java but slowly compared to Go. It uses more memory than you'd expect but less than you'd fear.

This chapter is about making your CLI tools fast—not just functionally fast, but perceptually fast. Because in the world of command-line tools, perception is reality.

## The Startup Performance Problem

Every millisecond of startup time is a papercut. Death by a thousand papercuts is real, and it starts with `require()`.

```javascript
// The problem: Every require() is synchronous and blocking
console.time('Startup');

const fs = require('fs');              // ~1ms
const path = require('path');          // ~0.5ms
const chalk = require('chalk');        // ~15ms
const commander = require('commander'); // ~8ms
const inquirer = require('inquirer');  // ~45ms (!!)
const lodash = require('lodash');      // ~25ms

console.timeEnd('Startup');  // Total: ~95ms just for imports!

// Your actual code hasn't even run yet
```

The solution? Lazy loading:

```javascript
// Fast startup with lazy loading
console.time('Startup');

// Core modules are cheap, load them immediately
import fs from 'fs';
import path from 'path';
import { performance } from 'perf_hooks';

class FastCLI {
  constructor() {
    this.modules = {};
    this.startupTime = performance.now();
  }

  // Lazy load expensive modules
  get chalk() {
    if (!this.modules.chalk) {
      this.modules.chalk = import('chalk');
    }
    return this.modules.chalk;
  }

  get inquirer() {
    if (!this.modules.inquirer) {
      this.modules.inquirer = import('inquirer');
    }
    return this.modules.inquirer;
  }

  async runCommand(command) {
    // Only load what you need, when you need it
    if (command === 'interactive') {
      const { default: inquirer } = await this.inquirer;
      // Now incur the 45ms cost
    } else {
      // Fast path - no inquirer needed
      console.log('Running fast command');
    }

    const loadTime = performance.now() - this.startupTime;
    console.log(`Ready in ${loadTime.toFixed(0)}ms`);
  }
}

console.timeEnd('Startup');  // Now only ~2ms!
```

## Module Bundling: The Nuclear Option

Sometimes lazy loading isn't enough. Enter bundling:

```javascript
// build.js - Bundle your CLI for production
import esbuild from 'esbuild';
import { nodeExternalsPlugin } from 'esbuild-node-externals';

async function bundle() {
  const result = await esbuild.build({
    entryPoints: ['src/cli.js'],
    bundle: true,
    platform: 'node',
    target: 'node14',
    outfile: 'dist/cli.js',
    minify: true,
    sourcemap: true,

    // Keep native modules external
    plugins: [nodeExternalsPlugin()],

    // Tree-shake unused code
    treeShaking: true,

    // Replace __dirname and __filename
    define: {
      '__dirname': JSON.stringify(process.cwd()),
      'process.env.NODE_ENV': '"production"'
    },

    // Bundle but keep these external
    external: ['fsevents', 'node-pty', 'cpu-features'],

    // Optimize for size
    metafile: true,
    analyze: true
  });

  // Analyze bundle size
  const text = await esbuild.analyzeMetafile(result.metafile);
  console.log(text);

  // Before: 500 files, 50MB, 300ms startup
  // After:  1 file, 2MB, 50ms startup
}

bundle();
```

## Caching: Remember Everything

The fastest operation is the one you don't do:

```javascript
import crypto from 'crypto';
import os from 'os';
import path from 'path';
import fs from 'fs/promises';

class SmartCache {
  constructor(options = {}) {
    this.cacheDir = options.cacheDir || path.join(os.homedir(), '.cli-cache');
    this.ttl = options.ttl || 3600000; // 1 hour default
    this.maxSize = options.maxSize || 100 * 1024 * 1024; // 100MB
    this.compression = options.compression || false;
    this.stats = {
      hits: 0,
      misses: 0,
      size: 0
    };

    this.ensureCacheDir();
  }

  async ensureCacheDir() {
    await fs.mkdir(this.cacheDir, { recursive: true });
  }

  generateKey(...args) {
    const hash = crypto.createHash('sha256');
    hash.update(JSON.stringify(args));
    return hash.digest('hex');
  }

  async get(key, options = {}) {
    const start = performance.now();
    const filepath = path.join(this.cacheDir, `${key}.json`);

    try {
      const stats = await fs.stat(filepath);

      // Check if cache is expired
      const age = Date.now() - stats.mtime.getTime();
      const ttl = options.ttl || this.ttl;

      if (age > ttl) {
        this.stats.misses++;
        return null;
      }

      const data = await fs.readFile(filepath, 'utf8');
      const parsed = JSON.parse(data);

      this.stats.hits++;
      const duration = performance.now() - start;

      if (duration > 10) {
        console.log(chalk.yellow(`Cache read took ${duration.toFixed(0)}ms`));
      }

      return parsed.value;
    } catch (error) {
      this.stats.misses++;
      return null;
    }
  }

  async set(key, value, options = {}) {
    const filepath = path.join(this.cacheDir, `${key}.json`);

    const data = {
      value,
      timestamp: Date.now(),
      metadata: options.metadata || {}
    };

    await fs.writeFile(filepath, JSON.stringify(data));

    // Update stats
    const stats = await fs.stat(filepath);
    this.stats.size += stats.size;

    // Cleanup if too large
    if (this.stats.size > this.maxSize) {
      await this.cleanup();
    }
  }

  async memoize(fn, options = {}) {
    return async (...args) => {
      const key = options.key || this.generateKey(fn.name, ...args);
      const cached = await this.get(key, options);

      if (cached !== null) {
        return cached;
      }

      const result = await fn(...args);
      await this.set(key, result, options);

      return result;
    };
  }

  async cleanup() {
    const files = await fs.readdir(this.cacheDir);
    const fileStats = await Promise.all(
      files.map(async (file) => {
        const filepath = path.join(this.cacheDir, file);
        const stats = await fs.stat(filepath);
        return { filepath, stats };
      })
    );

    // Sort by last modified time (LRU)
    fileStats.sort((a, b) => a.stats.mtime - b.stats.mtime);

    // Remove oldest files until under size limit
    let totalSize = fileStats.reduce((sum, f) => sum + f.stats.size, 0);

    for (const { filepath, stats } of fileStats) {
      if (totalSize <= this.maxSize * 0.8) break;  // Keep 80% full

      await fs.unlink(filepath);
      totalSize -= stats.size;
    }

    this.stats.size = totalSize;
  }

  getStats() {
    const hitRate = this.stats.hits / (this.stats.hits + this.stats.misses) * 100;
    return {
      ...this.stats,
      hitRate: hitRate.toFixed(1) + '%',
      sizeInMB: (this.stats.size / 1024 / 1024).toFixed(2) + 'MB'
    };
  }
}

// Using the cache
class GitOperations {
  constructor() {
    this.cache = new SmartCache({
      ttl: 5 * 60 * 1000  // 5 minutes for git operations
    });
  }

  async getGitLog(options = {}) {
    // Cache expensive git operations
    return this.cache.memoize(async () => {
      const { stdout } = await exec('git log --oneline -n 100');
      return stdout.split('\n');
    }, {
      key: `git-log-${JSON.stringify(options)}`,
      ttl: 60000  // 1 minute for logs
    })();
  }

  async getFileHistory(filepath) {
    // Cache per-file history
    return this.cache.memoize(async (file) => {
      const { stdout } = await exec(`git log --oneline -- ${file}`);
      return stdout.split('\n');
    })(filepath);
  }
}
```

## Profiling: Know Your Enemy

You can't optimize what you can't measure:

```javascript
import { performance, PerformanceObserver } from 'perf_hooks';
import v8 from 'v8';

class PerformanceProfiler {
  constructor() {
    this.marks = new Map();
    this.measures = new Map();
    this.enabled = process.env.PROFILE === 'true';

    if (this.enabled) {
      this.setupObserver();
    }
  }

  setupObserver() {
    const obs = new PerformanceObserver((items) => {
      items.getEntries().forEach((entry) => {
        if (entry.entryType === 'measure') {
          console.log(chalk.gray(
            `⏱  ${entry.name}: ${entry.duration.toFixed(2)}ms`
          ));
        }
      });
    });

    obs.observe({ entryTypes: ['measure'] });
  }

  mark(name) {
    if (!this.enabled) return;

    performance.mark(`${name}-start`);
    this.marks.set(name, performance.now());
  }

  measure(name) {
    if (!this.enabled) return;

    const start = this.marks.get(name);
    if (!start) return;

    const duration = performance.now() - start;
    performance.measure(name, `${name}-start`);

    if (!this.measures.has(name)) {
      this.measures.set(name, []);
    }

    this.measures.get(name).push(duration);
    this.marks.delete(name);

    return duration;
  }

  async profile(name, fn) {
    if (!this.enabled) return fn();

    this.mark(name);
    try {
      const result = await fn();
      return result;
    } finally {
      this.measure(name);
    }
  }

  getStats() {
    const stats = {};

    for (const [name, durations] of this.measures) {
      const sorted = durations.sort((a, b) => a - b);
      stats[name] = {
        count: durations.length,
        total: durations.reduce((a, b) => a + b, 0),
        avg: durations.reduce((a, b) => a + b, 0) / durations.length,
        min: sorted[0],
        max: sorted[sorted.length - 1],
        p50: sorted[Math.floor(sorted.length * 0.5)],
        p95: sorted[Math.floor(sorted.length * 0.95)],
        p99: sorted[Math.floor(sorted.length * 0.99)]
      };
    }

    return stats;
  }

  getMemoryUsage() {
    const usage = process.memoryUsage();
    return {
      rss: (usage.rss / 1024 / 1024).toFixed(2) + 'MB',
      heapTotal: (usage.heapTotal / 1024 / 1024).toFixed(2) + 'MB',
      heapUsed: (usage.heapUsed / 1024 / 1024).toFixed(2) + 'MB',
      external: (usage.external / 1024 / 1024).toFixed(2) + 'MB',
      arrayBuffers: (usage.arrayBuffers / 1024 / 1024).toFixed(2) + 'MB'
    };
  }

  getHeapStatistics() {
    const stats = v8.getHeapStatistics();
    return {
      totalHeapSize: (stats.total_heap_size / 1024 / 1024).toFixed(2) + 'MB',
      usedHeapSize: (stats.used_heap_size / 1024 / 1024).toFixed(2) + 'MB',
      heapSizeLimit: (stats.heap_size_limit / 1024 / 1024).toFixed(2) + 'MB',
      mallocedMemory: (stats.malloced_memory / 1024 / 1024).toFixed(2) + 'MB'
    };
  }

  writeHeapSnapshot(filename = 'heap.heapsnapshot') {
    if (!this.enabled) return;

    const heapSnapshot = v8.writeHeapSnapshot();
    console.log(chalk.green(`Heap snapshot written to ${filename}`));
    return heapSnapshot;
  }

  report() {
    if (!this.enabled) return;

    console.log(chalk.bold('\n📊 Performance Report\n'));

    const stats = this.getStats();
    for (const [name, data] of Object.entries(stats)) {
      console.log(chalk.yellow(name));
      console.log(`  Calls: ${data.count}`);
      console.log(`  Avg: ${data.avg.toFixed(2)}ms`);
      console.log(`  P95: ${data.p95.toFixed(2)}ms`);
      console.log(`  Total: ${data.total.toFixed(2)}ms`);
    }

    console.log(chalk.yellow('\nMemory Usage:'));
    const memory = this.getMemoryUsage();
    for (const [key, value] of Object.entries(memory)) {
      console.log(`  ${key}: ${value}`);
    }
  }
}

// Using the profiler
const profiler = new PerformanceProfiler();

async function processFiles(files) {
  return profiler.profile('processFiles', async () => {
    for (const file of files) {
      await profiler.profile('readFile', () => fs.readFile(file));
      await profiler.profile('transform', () => transformFile(file));
      await profiler.profile('writeFile', () => fs.writeFile(file));
    }
  });
}

// At the end of your program
process.on('exit', () => {
  profiler.report();
});
```

## Concurrency: Do More at Once

Node.js is single-threaded, but that doesn't mean you can't do multiple things:

```javascript
class ConcurrencyManager {
  constructor(limit = 5) {
    this.limit = limit;
    this.running = 0;
    this.queue = [];
  }

  async run(fn) {
    while (this.running >= this.limit) {
      await new Promise(resolve => this.queue.push(resolve));
    }

    this.running++;

    try {
      return await fn();
    } finally {
      this.running--;

      const resolve = this.queue.shift();
      if (resolve) resolve();
    }
  }

  async map(items, fn) {
    return Promise.all(items.map(item => this.run(() => fn(item))));
  }

  async batch(items, batchSize, fn) {
    const results = [];

    for (let i = 0; i < items.length; i += batchSize) {
      const batch = items.slice(i, i + batchSize);
      const batchResults = await this.map(batch, fn);
      results.push(...batchResults);
    }

    return results;
  }
}

// Practical example: Processing files
class FileProcessor {
  constructor() {
    this.concurrency = new ConcurrencyManager(10);  // Process 10 files at once
  }

  async processDirectory(dir) {
    const files = await fs.readdir(dir);

    console.time('Processing');

    // Slow: Process one by one
    // for (const file of files) {
    //   await this.processFile(file);
    // }
    // Time: 10 seconds for 100 files

    // Fast: Process concurrently
    await this.concurrency.map(files, file => this.processFile(file));
    // Time: 2 seconds for 100 files

    console.timeEnd('Processing');
  }

  async processFile(file) {
    const data = await fs.readFile(file);
    const processed = await this.transform(data);
    await fs.writeFile(file + '.processed', processed);
  }
}
```

## Worker Threads: True Parallelism

For CPU-intensive tasks, worker threads are your friend:

```javascript
import { Worker, isMainThread, parentPort, workerData } from 'worker_threads';
import os from 'os';

class WorkerPool {
  constructor(workerPath, poolSize = os.cpus().length) {
    this.workerPath = workerPath;
    this.poolSize = poolSize;
    this.workers = [];
    this.freeWorkers = [];
    this.queue = [];

    this.init();
  }

  init() {
    for (let i = 0; i < this.poolSize; i++) {
      this.addNewWorker();
    }
  }

  addNewWorker() {
    const worker = new Worker(this.workerPath);

    worker.on('message', (result) => {
      worker.currentResolve(result);
      worker.currentResolve = null;

      this.freeWorkers.push(worker);
      this.processNext();
    });

    worker.on('error', (err) => {
      if (worker.currentReject) {
        worker.currentReject(err);
      }
    });

    this.workers.push(worker);
    this.freeWorkers.push(worker);
  }

  async execute(data) {
    return new Promise((resolve, reject) => {
      this.queue.push({ data, resolve, reject });
      this.processNext();
    });
  }

  processNext() {
    if (this.queue.length === 0 || this.freeWorkers.length === 0) {
      return;
    }

    const { data, resolve, reject } = this.queue.shift();
    const worker = this.freeWorkers.shift();

    worker.currentResolve = resolve;
    worker.currentReject = reject;
    worker.postMessage(data);
  }

  async terminate() {
    await Promise.all(this.workers.map(w => w.terminate()));
  }
}

// worker.js
if (!isMainThread) {
  parentPort.on('message', async (data) => {
    try {
      // CPU-intensive operation
      const result = await processHeavyComputation(data);
      parentPort.postMessage(result);
    } catch (error) {
      throw error;
    }
  });

  async function processHeavyComputation(data) {
    // Example: Image processing, crypto, compression
    let result = 0;
    for (let i = 0; i < 100000000; i++) {
      result += Math.sqrt(i) * Math.random();
    }
    return result;
  }
}

// Using the worker pool
async function processDataInParallel(items) {
  const pool = new WorkerPool('./worker.js');

  const startSingle = performance.now();
  // Single-threaded
  // for (const item of items) {
  //   await processHeavyComputation(item);
  // }
  // Time: 10000ms

  const startParallel = performance.now();
  // Multi-threaded
  const results = await Promise.all(
    items.map(item => pool.execute(item))
  );
  const parallelTime = performance.now() - startParallel;
  // Time: 2500ms on 4-core machine

  await pool.terminate();
  return results;
}
```

## Memory Management: Don't Be Greedy

Memory leaks in CLI tools are especially painful because they accumulate over long-running operations:

```javascript
class MemoryManager {
  constructor(options = {}) {
    this.maxHeapUsage = options.maxHeapUsage || 500 * 1024 * 1024; // 500MB
    this.checkInterval = options.checkInterval || 10000; // 10 seconds
    this.gcInterval = options.gcInterval || 60000; // 1 minute

    this.startMonitoring();
  }

  startMonitoring() {
    // Monitor memory usage
    this.memoryCheckInterval = setInterval(() => {
      const usage = process.memoryUsage();

      if (usage.heapUsed > this.maxHeapUsage) {
        console.warn(chalk.yellow(`⚠️  High memory usage: ${
          (usage.heapUsed / 1024 / 1024).toFixed(0)
        }MB`));

        // Force garbage collection if available
        if (global.gc) {
          global.gc();
          console.log(chalk.gray('Forced garbage collection'));
        }
      }
    }, this.checkInterval);

    // Periodic garbage collection
    if (global.gc) {
      this.gcInterval = setInterval(() => {
        global.gc();
      }, this.gcInterval);
    }
  }

  stop() {
    clearInterval(this.memoryCheckInterval);
    clearInterval(this.gcInterval);
  }

  // Prevent memory leaks from event emitters
  safeListen(emitter, event, handler, options = {}) {
    const {
      maxListeners = 10,
      once = false
    } = options;

    // Check for potential memory leak
    const currentListeners = emitter.listenerCount(event);
    if (currentListeners > maxListeners) {
      console.warn(chalk.yellow(
        `⚠️  Potential memory leak: ${currentListeners} listeners for '${event}'`
      ));
    }

    if (once) {
      emitter.once(event, handler);
    } else {
      emitter.on(event, handler);
    }

    // Return cleanup function
    return () => emitter.removeListener(event, handler);
  }

  // Stream data instead of loading into memory
  async processLargeFile(filepath, processLine) {
    return new Promise((resolve, reject) => {
      const stream = fs.createReadStream(filepath, {
        encoding: 'utf8',
        highWaterMark: 16 * 1024  // 16KB chunks
      });

      const rl = readline.createInterface({
        input: stream,
        crlfDelay: Infinity
      });

      let lineNumber = 0;

      rl.on('line', async (line) => {
        lineNumber++;
        await processLine(line, lineNumber);
      });

      rl.on('close', resolve);
      rl.on('error', reject);

      // Prevent memory leaks
      stream.on('error', reject);
    });
  }

  // Use object pools for frequently created objects
  createPool(factory, reset, maxSize = 100) {
    const pool = [];

    return {
      acquire() {
        return pool.pop() || factory();
      },

      release(obj) {
        if (pool.length < maxSize) {
          reset(obj);
          pool.push(obj);
        }
      },

      size() {
        return pool.length;
      }
    };
  }
}

// Example: Buffer pool for file operations
const bufferPool = new MemoryManager().createPool(
  () => Buffer.allocUnsafe(64 * 1024),  // Create 64KB buffer
  (buffer) => buffer.fill(0),           // Reset buffer
  50                                     // Keep max 50 buffers
);

async function processWithPooledBuffers(files) {
  for (const file of files) {
    const buffer = bufferPool.acquire();

    try {
      await processFile(file, buffer);
    } finally {
      bufferPool.release(buffer);
    }
  }
}
```

## Network Optimization: Be a Good Citizen

Network operations are often the bottleneck in CLI tools:

```javascript
import http from 'http';
import https from 'https';
import { URL } from 'url';

class NetworkOptimizer {
  constructor(options = {}) {
    this.maxSockets = options.maxSockets || 50;
    this.timeout = options.timeout || 30000;
    this.retries = options.retries || 3;
    this.cache = new Map();

    // Configure global agents
    this.httpAgent = new http.Agent({
      keepAlive: true,
      keepAliveMsecs: 1000,
      maxSockets: this.maxSockets,
      maxFreeSockets: 10
    });

    this.httpsAgent = new https.Agent({
      keepAlive: true,
      keepAliveMsecs: 1000,
      maxSockets: this.maxSockets,
      maxFreeSockets: 10
    });
  }

  async fetch(url, options = {}) {
    const {
      method = 'GET',
      headers = {},
      body = null,
      cache = true
    } = options;

    // Check cache for GET requests
    if (method === 'GET' && cache) {
      const cached = this.cache.get(url);
      if (cached && Date.now() - cached.timestamp < 60000) {
        return cached.data;
      }
    }

    // Retry logic with exponential backoff
    for (let attempt = 0; attempt <= this.retries; attempt++) {
      try {
        const response = await this.makeRequest(url, {
          method,
          headers,
          body,
          timeout: this.timeout,
          agent: url.startsWith('https') ? this.httpsAgent : this.httpAgent
        });

        // Cache successful GET responses
        if (method === 'GET' && cache) {
          this.cache.set(url, {
            data: response,
            timestamp: Date.now()
          });
        }

        return response;
      } catch (error) {
        if (attempt === this.retries) throw error;

        const delay = Math.pow(2, attempt) * 1000;
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
  }

  async parallelFetch(urls, options = {}) {
    const {
      concurrency = 10,
      onProgress = null
    } = options;

    const results = new Array(urls.length);
    const queue = urls.map((url, index) => ({ url, index }));
    const inProgress = [];

    while (queue.length > 0 || inProgress.length > 0) {
      // Start new requests up to concurrency limit
      while (queue.length > 0 && inProgress.length < concurrency) {
        const { url, index } = queue.shift();

        const promise = this.fetch(url)
          .then(data => {
            results[index] = data;
            if (onProgress) {
              onProgress(index, urls.length);
            }
          })
          .catch(error => {
            results[index] = { error: error.message };
          })
          .finally(() => {
            const idx = inProgress.indexOf(promise);
            if (idx !== -1) inProgress.splice(idx, 1);
          });

        inProgress.push(promise);
      }

      // Wait for at least one to complete
      if (inProgress.length > 0) {
        await Promise.race(inProgress);
      }
    }

    return results;
  }

  makeRequest(url, options) {
    return new Promise((resolve, reject) => {
      const parsedUrl = new URL(url);
      const module = parsedUrl.protocol === 'https:' ? https : http;

      const req = module.request(parsedUrl, options, (res) => {
        let data = '';

        res.on('data', chunk => {
          data += chunk;
        });

        res.on('end', () => {
          if (res.statusCode >= 200 && res.statusCode < 300) {
            resolve(data);
          } else {
            reject(new Error(`HTTP ${res.statusCode}: ${data}`));
          }
        });
      });

      req.on('error', reject);
      req.on('timeout', () => {
        req.destroy();
        reject(new Error('Request timeout'));
      });

      if (options.body) {
        req.write(options.body);
      }

      req.end();
    });
  }
}

// Usage
const network = new NetworkOptimizer();

// Fetch 100 URLs efficiently
const urls = Array.from({ length: 100 }, (_, i) =>
  `https://api.example.com/data/${i}`
);

const results = await network.parallelFetch(urls, {
  concurrency: 20,
  onProgress: (current, total) => {
    process.stdout.write(`\rProgress: ${current}/${total}`);
  }
});
```

## Optimization Strategies: The Toolbox

Here's a comprehensive optimization strategy:

```javascript
class OptimizationStrategy {
  // 1. Debounce expensive operations
  debounce(func, wait) {
    let timeout;
    return function executedFunction(...args) {
      const later = () => {
        clearTimeout(timeout);
        func(...args);
      };
      clearTimeout(timeout);
      timeout = setTimeout(later, wait);
    };
  }

  // 2. Throttle rapid operations
  throttle(func, limit) {
    let inThrottle;
    return function(...args) {
      if (!inThrottle) {
        func.apply(this, args);
        inThrottle = true;
        setTimeout(() => inThrottle = false, limit);
      }
    };
  }

  // 3. Batch operations
  createBatcher(processor, options = {}) {
    const {
      maxBatchSize = 100,
      maxWaitTime = 100
    } = options;

    let batch = [];
    let timeout;

    const processBatch = async () => {
      if (batch.length === 0) return;

      const currentBatch = batch;
      batch = [];
      clearTimeout(timeout);

      await processor(currentBatch);
    };

    return {
      add(item) {
        batch.push(item);

        if (batch.length >= maxBatchSize) {
          processBatch();
        } else if (!timeout) {
          timeout = setTimeout(processBatch, maxWaitTime);
        }
      },

      flush() {
        return processBatch();
      }
    };
  }

  // 4. Use binary search for sorted data
  binarySearch(arr, target, compare = (a, b) => a - b) {
    let left = 0;
    let right = arr.length - 1;

    while (left <= right) {
      const mid = Math.floor((left + right) / 2);
      const comparison = compare(arr[mid], target);

      if (comparison === 0) return mid;
      if (comparison < 0) left = mid + 1;
      else right = mid - 1;
    }

    return -1;
  }

  // 5. Use tries for prefix matching
  createTrie() {
    const root = {};

    return {
      insert(word) {
        let node = root;
        for (const char of word) {
          if (!node[char]) {
            node[char] = {};
          }
          node = node[char];
        }
        node.isEnd = true;
      },

      search(prefix) {
        let node = root;
        for (const char of prefix) {
          if (!node[char]) return false;
          node = node[char];
        }
        return true;
      },

      autocomplete(prefix) {
        let node = root;
        for (const char of prefix) {
          if (!node[char]) return [];
          node = node[char];
        }

        const results = [];
        const dfs = (currentNode, currentWord) => {
          if (currentNode.isEnd) {
            results.push(prefix + currentWord);
          }
          for (const [char, nextNode] of Object.entries(currentNode)) {
            if (char !== 'isEnd') {
              dfs(nextNode, currentWord + char);
            }
          }
        };

        dfs(node, '');
        return results;
      }
    };
  }

  // 6. Optimize string operations
  createStringBuilder() {
    const parts = [];

    return {
      append(str) {
        parts.push(str);
        return this;
      },

      toString() {
        return parts.join('');
      }
    };
  }
}
```

## Real-World Optimization: A Case Study

Let's optimize a real file search tool:

```javascript
// Before: Slow file search
class SlowFileSearch {
  async search(directory, pattern) {
    const results = [];

    async function walk(dir) {
      const files = await fs.readdir(dir);

      for (const file of files) {
        const filepath = path.join(dir, file);
        const stats = await fs.stat(filepath);

        if (stats.isDirectory()) {
          await walk(filepath);  // Recursive, sequential
        } else if (file.includes(pattern)) {
          results.push(filepath);
        }
      }
    }

    await walk(directory);
    return results;
  }
}

// After: Fast file search
class FastFileSearch {
  constructor() {
    this.cache = new SmartCache();
    this.concurrency = new ConcurrencyManager(50);
    this.profiler = new PerformanceProfiler();
  }

  async search(directory, pattern, options = {}) {
    return this.profiler.profile('search', async () => {
      // Check cache first
      const cacheKey = `search:${directory}:${pattern}`;
      const cached = await this.cache.get(cacheKey);
      if (cached) return cached;

      const results = [];
      const queue = [directory];
      const visited = new Set();

      // Use glob for better performance
      if (options.useGlob) {
        const globPattern = path.join(directory, '**', `*${pattern}*`);
        const files = await glob(globPattern, {
          ignore: ['**/node_modules/**', '**/.git/**']
        });
        return files;
      }

      // Parallel directory processing
      while (queue.length > 0) {
        const batch = queue.splice(0, 10);

        await this.concurrency.map(batch, async (dir) => {
          if (visited.has(dir)) return;
          visited.add(dir);

          const entries = await fs.readdir(dir, { withFileTypes: true });

          // Process entries in parallel
          await Promise.all(entries.map(async (entry) => {
            const filepath = path.join(dir, entry.name);

            if (entry.isDirectory()) {
              // Skip hidden and system directories
              if (!entry.name.startsWith('.') &&
                  entry.name !== 'node_modules') {
                queue.push(filepath);
              }
            } else if (this.matches(entry.name, pattern)) {
              results.push(filepath);
            }
          }));
        });
      }

      // Cache results
      await this.cache.set(cacheKey, results, { ttl: 60000 });

      return results;
    });
  }

  matches(filename, pattern) {
    // Use optimized matching
    if (pattern.includes('*')) {
      // Convert glob to regex
      const regex = new RegExp(
        pattern.replace(/\*/g, '.*').replace(/\?/g, '.')
      );
      return regex.test(filename);
    }

    return filename.includes(pattern);
  }
}

// Performance comparison
async function benchmark() {
  const slow = new SlowFileSearch();
  const fast = new FastFileSearch();

  console.time('Slow Search');
  await slow.search('/large/directory', 'test');
  console.timeEnd('Slow Search');  // 15000ms

  console.time('Fast Search');
  await fast.search('/large/directory', 'test');
  console.timeEnd('Fast Search');  // 500ms

  console.time('Fast Search (Cached)');
  await fast.search('/large/directory', 'test');
  console.timeEnd('Fast Search (Cached)');  // 5ms
}
```

## The Performance Mindset

Performance optimization is not about making everything fast—it's about making the right things fast. Users don't care if your internal sorting algorithm is O(n log n) or O(n²) if they're sorting 10 items. They care about:

1. **Startup time**: How fast your tool is ready to work
2. **Responsiveness**: How quickly it reacts to input
3. **Progress feedback**: Knowing something is happening
4. **Completion time**: How long the actual task takes

The key insights:

- **Lazy load everything**: Don't pay for what you don't use
- **Cache aggressively**: The fastest operation is no operation
- **Parallelize when possible**: Use all available resources
- **Stream, don't buffer**: Process data as it arrives
- **Profile, don't guess**: Measure before optimizing

Remember: premature optimization is the root of all evil, but mature optimization is the fruit of all good. Know when you're dealing with each.

In the next chapter, we'll tackle testing and debugging, because fast code that doesn't work is just failing quickly. We'll learn how to test CLI tools, debug them when they inevitably break, and ensure they keep working as they evolve.