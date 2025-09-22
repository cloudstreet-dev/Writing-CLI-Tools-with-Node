# Chapter 5: Working with Files and the Filesystem

Every CLI tool eventually becomes a glorified file manager. You might start with noble intentions—"I'm building an AI assistant" or "I'm creating a blockchain validator"—but give it a few weeks, and you'll be neck-deep in file I/O, wrestling with path separators, and wondering why Windows insists on using backslashes like some kind of rebel.

The filesystem is where CLI tools live and breathe. It's their natural habitat. And Node.js, with its asynchronous I/O and streaming capabilities, is surprisingly good at it—once you learn to navigate the minefield of callbacks, promises, streams, and the eternal question of whether to use `fs` or `fs/promises` or `fs-extra` or just give up and use `child_process.exec('cat file.txt')`.

## The Evolution of Node.js File Operations

Let's start with a history lesson, because understanding where we've been helps us appreciate where we are.

```javascript
// The Dark Ages: Callback Hell (Node.js v0.x - v6)
const fs = require('fs');

fs.readFile('input.txt', 'utf8', function(err, data) {
  if (err) {
    console.error('Error reading file:', err);
    return;
  }

  const processed = data.toUpperCase();

  fs.writeFile('output.txt', processed, function(err) {
    if (err) {
      console.error('Error writing file:', err);
      return;
    }

    fs.stat('output.txt', function(err, stats) {
      if (err) {
        console.error('Error getting file stats:', err);
        return;
      }

      console.log('File size:', stats.size);
      // Welcome to callback hell. Population: you.
    });
  });
});
```

```javascript
// The Renaissance: Promises (Node.js v10+)
const fs = require('fs').promises;

async function processFile() {
  try {
    const data = await fs.readFile('input.txt', 'utf8');
    const processed = data.toUpperCase();
    await fs.writeFile('output.txt', processed);
    const stats = await fs.stat('output.txt');
    console.log('File size:', stats.size);
  } catch (err) {
    console.error('Error:', err);
  }
}

// Ah, sweet linear flow. This is what happiness looks like.
```

```javascript
// The Modern Era: Multiple Approaches (Node.js v14+)
import { readFile, writeFile } from 'fs/promises';
import { createReadStream, createWriteStream } from 'fs';
import { pipeline } from 'stream/promises';

// For small files
const data = await readFile('small.txt', 'utf8');

// For large files
await pipeline(
  createReadStream('huge.txt'),
  transformStream,
  createWriteStream('output.txt')
);

// For when you hate yourself
import { readFileSync } from 'fs';
const data = readFileSync('why.txt', 'utf8');  // Blocks everything. Why would you do this?
```

## The Art of Path Manipulation

Paths are where cross-platform compatibility goes to die. Windows uses `\`, Unix uses `/`, and somewhere, a developer is crying because their CLI tool just tried to open `C:\Users\bobsprojects\my-app\src\index.js` on a Mac.

```javascript
import path from 'path';
import { fileURLToPath } from 'url';
import { dirname } from 'path';

// The cross-platform way
const filePath = path.join('src', 'components', 'Header.js');
// Results: 'src/components/Header.js' on Unix
//          'src\components\Header.js' on Windows

// Getting the current directory in ESM modules
const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

// Building paths safely
const configPath = path.join(__dirname, '..', 'config', 'app.json');

// Parsing paths
const parsed = path.parse('/users/bob/documents/report.pdf');
console.log(parsed);
// {
//   root: '/',
//   dir: '/users/bob/documents',
//   base: 'report.pdf',
//   ext: '.pdf',
//   name: 'report'
// }

// Resolving relative paths
const absolute = path.resolve('../../file.txt');
// Turns relative paths into absolute ones

// Normalizing paths (fixing the mess)
const messy = '/users//bob/../bob/./documents//report.pdf';
const clean = path.normalize(messy);
// Result: '/users/bob/documents/report.pdf'

// Getting relative paths
const from = '/users/bob/projects/app';
const to = '/users/bob/documents/file.txt';
const relative = path.relative(from, to);
// Result: '../../documents/file.txt'
```

## File System Watching: The Real-Time Revolution

Modern CLI tools don't just read files; they watch them, like a protective parent watching their teenager's social media. This is how tools like nodemon, webpack-dev-server, and every hot-reload system work.

```javascript
import { watch, watchFile } from 'fs';
import chokidar from 'chokidar';  // Because native watching is pain

// Native Node.js watching (prepare for disappointment)
watch('./src', { recursive: true }, (eventType, filename) => {
  console.log(`Event: ${eventType}, File: ${filename}`);
  // Problems:
  // 1. 'filename' might be null
  // 2. Multiple events for single change
  // 3. Doesn't work recursively on Linux
  // 4. Generally unreliable
});

// The better way: chokidar
class FileWatcher {
  constructor() {
    this.watchers = new Map();
  }

  watch(pattern, options = {}) {
    const watcher = chokidar.watch(pattern, {
      ignored: /(^|[\/\\])\../,  // Ignore dotfiles
      persistent: true,
      ignoreInitial: true,
      awaitWriteFinish: {
        stabilityThreshold: 300,
        pollInterval: 100
      },
      ...options
    });

    watcher
      .on('add', path => this.handleAdd(path))
      .on('change', path => this.handleChange(path))
      .on('unlink', path => this.handleRemove(path))
      .on('addDir', path => this.handleAddDir(path))
      .on('unlinkDir', path => this.handleRemoveDir(path))
      .on('error', error => this.handleError(error))
      .on('ready', () => this.handleReady());

    this.watchers.set(pattern, watcher);
    return watcher;
  }

  handleChange(filePath) {
    console.log(chalk.yellow(`✎ Modified: ${filePath}`));

    // Debounce rapid changes
    if (this.changeTimeout) {
      clearTimeout(this.changeTimeout);
    }

    this.changeTimeout = setTimeout(() => {
      this.processChange(filePath);
    }, 100);
  }

  async processChange(filePath) {
    const ext = path.extname(filePath);

    switch (ext) {
      case '.js':
      case '.ts':
        console.log('🔄 Reloading application...');
        await this.reloadApp();
        break;
      case '.css':
        console.log('🎨 Hot-reloading styles...');
        await this.hotReloadCSS(filePath);
        break;
      case '.json':
        console.log('⚙️  Reloading configuration...');
        await this.reloadConfig(filePath);
        break;
    }
  }

  handleAdd(filePath) {
    console.log(chalk.green(`✚ Added: ${filePath}`));
  }

  handleRemove(filePath) {
    console.log(chalk.red(`✖ Deleted: ${filePath}`));
  }

  handleError(error) {
    console.error(chalk.red(`Watch error: ${error}`));
  }

  handleReady() {
    console.log(chalk.green('👁️  File watcher ready'));
  }

  stopWatching(pattern) {
    const watcher = this.watchers.get(pattern);
    if (watcher) {
      watcher.close();
      this.watchers.delete(pattern);
    }
  }

  stopAll() {
    for (const watcher of this.watchers.values()) {
      watcher.close();
    }
    this.watchers.clear();
  }
}
```

## Streaming: The Memory-Efficient Way

When your CLI tool needs to process a 10GB log file, loading it all into memory isn't an option (unless you enjoy watching your computer swap itself to death). This is where streams shine.

```javascript
import { createReadStream, createWriteStream } from 'fs';
import { Transform, pipeline } from 'stream';
import { promisify } from 'util';
import readline from 'readline';
import zlib from 'zlib';

// Processing large files line by line
async function processLargeFile(inputPath) {
  const fileStream = createReadStream(inputPath);
  const rl = readline.createInterface({
    input: fileStream,
    crlfDelay: Infinity  // Handle Windows line endings
  });

  let lineNumber = 0;
  let errorCount = 0;

  for await (const line of rl) {
    lineNumber++;

    if (line.includes('ERROR')) {
      errorCount++;
      console.log(chalk.red(`Line ${lineNumber}: ${line}`));
    }

    // Show progress every 10,000 lines
    if (lineNumber % 10000 === 0) {
      process.stdout.write(`\rProcessed ${lineNumber} lines...`);
    }
  }

  console.log(`\n\nTotal errors found: ${errorCount}`);
}

// Custom transform stream
class CSVToJSON extends Transform {
  constructor(options = {}) {
    super(options);
    this.headers = null;
    this.lineNumber = 0;
  }

  _transform(chunk, encoding, callback) {
    const lines = chunk.toString().split('\n');

    for (const line of lines) {
      if (!line.trim()) continue;

      this.lineNumber++;

      if (this.lineNumber === 1) {
        this.headers = line.split(',').map(h => h.trim());
        continue;
      }

      const values = line.split(',').map(v => v.trim());
      const obj = {};

      this.headers.forEach((header, index) => {
        obj[header] = values[index] || null;
      });

      this.push(JSON.stringify(obj) + '\n');
    }

    callback();
  }
}

// Streaming pipeline with compression
async function compressFile(inputPath, outputPath) {
  const pipeline = promisify(stream.pipeline);

  await pipeline(
    createReadStream(inputPath),
    zlib.createGzip(),
    createWriteStream(outputPath + '.gz')
  );

  const inputStats = await fs.stat(inputPath);
  const outputStats = await fs.stat(outputPath + '.gz');

  const compression = ((1 - outputStats.size / inputStats.size) * 100).toFixed(2);
  console.log(chalk.green(`✓ Compressed: ${compression}% reduction`));
}

// Backpressure handling
class SmartFileProcessor {
  async process(inputPath, outputPath) {
    const readStream = createReadStream(inputPath, {
      highWaterMark: 16 * 1024  // 16KB chunks
    });

    const writeStream = createWriteStream(outputPath, {
      highWaterMark: 16 * 1024
    });

    const transform = new Transform({
      transform(chunk, encoding, callback) {
        // Simulate heavy processing
        setTimeout(() => {
          const processed = chunk.toString().toUpperCase();
          callback(null, processed);
        }, 10);
      }
    });

    // Handle backpressure properly
    readStream
      .pipe(transform)
      .on('pipe', () => console.log('Processing started'))
      .pipe(writeStream)
      .on('drain', () => console.log('Buffer drained, resuming'))
      .on('finish', () => console.log('Processing complete'));

    // Monitor progress
    let bytes = 0;
    readStream.on('data', chunk => {
      bytes += chunk.length;
      process.stdout.write(`\rProcessed: ${(bytes / 1024 / 1024).toFixed(2)} MB`);
    });
  }
}
```

## Glob Patterns: Finding Files Like a Pro

Real CLI tools don't make users type exact file paths. They accept patterns, wildcards, and probably telepathic input if we could figure out the API.

```javascript
import glob from 'glob';
import { glob as globPromise } from 'glob';
import micromatch from 'micromatch';
import fg from 'fast-glob';

class FileFinder {
  async findFiles(patterns, options = {}) {
    const defaultOptions = {
      ignore: ['**/node_modules/**', '**/.git/**', '**/dist/**'],
      dot: false,  // Include dotfiles?
      absolute: false,  // Return absolute paths?
      onlyFiles: true,  // Exclude directories?
      ...options
    };

    // Using fast-glob (fastest option)
    const files = await fg(patterns, defaultOptions);

    return this.processResults(files, options);
  }

  processResults(files, options) {
    // Sort by various criteria
    if (options.sortBy === 'size') {
      return this.sortBySize(files);
    } else if (options.sortBy === 'modified') {
      return this.sortByModified(files);
    } else if (options.sortBy === 'name') {
      return files.sort();
    }

    return files;
  }

  async sortBySize(files) {
    const withStats = await Promise.all(
      files.map(async file => ({
        path: file,
        size: (await fs.stat(file)).size
      }))
    );

    return withStats
      .sort((a, b) => b.size - a.size)
      .map(f => f.path);
  }

  async sortByModified(files) {
    const withStats = await Promise.all(
      files.map(async file => ({
        path: file,
        mtime: (await fs.stat(file)).mtime
      }))
    );

    return withStats
      .sort((a, b) => b.mtime - a.mtime)
      .map(f => f.path);
  }

  // Interactive file selection
  async selectFiles(pattern) {
    const spinner = ora('Searching for files...').start();
    const files = await this.findFiles(pattern);
    spinner.stop();

    if (files.length === 0) {
      console.log(chalk.yellow('No files found matching pattern'));
      return [];
    }

    if (files.length === 1) {
      console.log(chalk.green(`Found: ${files[0]}`));
      return files;
    }

    // Multiple files found, let user select
    const { selected } = await inquirer.prompt([
      {
        type: 'checkbox',
        name: 'selected',
        message: `Found ${files.length} files. Select which to process:`,
        choices: files.map(f => ({
          name: this.formatFilePath(f),
          value: f,
          checked: true
        })),
        pageSize: 15
      }
    ]);

    return selected;
  }

  formatFilePath(filePath) {
    const relative = path.relative(process.cwd(), filePath);
    const stats = fs.statSync(filePath);
    const size = this.formatFileSize(stats.size);
    const modified = this.formatDate(stats.mtime);

    return `${relative} ${chalk.gray(`(${size}, ${modified})`)}`;
  }

  formatFileSize(bytes) {
    const units = ['B', 'KB', 'MB', 'GB'];
    let size = bytes;
    let unitIndex = 0;

    while (size > 1024 && unitIndex < units.length - 1) {
      size /= 1024;
      unitIndex++;
    }

    return `${size.toFixed(1)}${units[unitIndex]}`;
  }

  formatDate(date) {
    const now = new Date();
    const diff = now - date;
    const days = Math.floor(diff / (1000 * 60 * 60 * 24));

    if (days === 0) return 'today';
    if (days === 1) return 'yesterday';
    if (days < 7) return `${days} days ago`;
    if (days < 30) return `${Math.floor(days / 7)} weeks ago`;
    if (days < 365) return `${Math.floor(days / 30)} months ago`;
    return `${Math.floor(days / 365)} years ago`;
  }
}

// Usage
const finder = new FileFinder();
const files = await finder.selectFiles('src/**/*.js');
console.log(`Processing ${files.length} files...`);
```

## File System Operations: Beyond Reading and Writing

There's more to file operations than just reading and writing. Let's explore the full toolkit:

```javascript
import { promises as fs } from 'fs';
import { promisify } from 'util';
import { exec } from 'child_process';
const execAsync = promisify(exec);

class FileOperations {
  async copy(source, destination, options = {}) {
    const {
      overwrite = false,
      preserveTimestamps = true,
      filter = () => true
    } = options;

    // Check if source exists
    const sourceStats = await fs.stat(source);

    // Check if destination exists
    let destExists = false;
    try {
      await fs.access(destination);
      destExists = true;
    } catch (err) {
      // Destination doesn't exist, which is fine
    }

    if (destExists && !overwrite) {
      throw new Error(`Destination already exists: ${destination}`);
    }

    if (sourceStats.isDirectory()) {
      await this.copyDirectory(source, destination, options);
    } else {
      await this.copyFile(source, destination, options);
    }
  }

  async copyDirectory(source, destination, options) {
    // Create destination directory
    await fs.mkdir(destination, { recursive: true });

    // Read source directory
    const entries = await fs.readdir(source, { withFileTypes: true });

    // Copy each entry
    for (const entry of entries) {
      const sourcePath = path.join(source, entry.name);
      const destPath = path.join(destination, entry.name);

      if (!options.filter(sourcePath)) continue;

      if (entry.isDirectory()) {
        await this.copyDirectory(sourcePath, destPath, options);
      } else {
        await this.copyFile(sourcePath, destPath, options);
      }
    }

    // Preserve timestamps if requested
    if (options.preserveTimestamps) {
      const stats = await fs.stat(source);
      await fs.utimes(destination, stats.atime, stats.mtime);
    }
  }

  async copyFile(source, destination, options) {
    await fs.copyFile(source, destination);

    if (options.preserveTimestamps) {
      const stats = await fs.stat(source);
      await fs.utimes(destination, stats.atime, stats.mtime);
    }
  }

  async move(source, destination, options = {}) {
    try {
      // Try rename first (fast, atomic)
      await fs.rename(source, destination);
    } catch (err) {
      // If rename fails (cross-device), copy then delete
      if (err.code === 'EXDEV') {
        await this.copy(source, destination, options);
        await this.remove(source);
      } else {
        throw err;
      }
    }
  }

  async remove(path, options = { recursive: true, force: true }) {
    try {
      const stats = await fs.stat(path);

      if (stats.isDirectory() && options.recursive) {
        await fs.rm(path, { recursive: true, force: options.force });
      } else {
        await fs.unlink(path);
      }
    } catch (err) {
      if (err.code === 'ENOENT' && options.force) {
        // File doesn't exist, and force is true, so we're good
        return;
      }
      throw err;
    }
  }

  async ensureDir(dirPath) {
    await fs.mkdir(dirPath, { recursive: true });
  }

  async emptyDir(dirPath) {
    const entries = await fs.readdir(dirPath, { withFileTypes: true });

    for (const entry of entries) {
      const fullPath = path.join(dirPath, entry.name);

      if (entry.isDirectory()) {
        await this.remove(fullPath, { recursive: true });
      } else {
        await fs.unlink(fullPath);
      }
    }
  }

  async touch(filePath, options = {}) {
    const {
      time = new Date(),
      createIfNotExists = true
    } = options;

    try {
      // Try to update timestamps
      await fs.utimes(filePath, time, time);
    } catch (err) {
      if (err.code === 'ENOENT' && createIfNotExists) {
        // Create empty file
        const fd = await fs.open(filePath, 'w');
        await fd.close();
        await fs.utimes(filePath, time, time);
      } else {
        throw err;
      }
    }
  }
}
```

## Temporary Files: The Unsung Heroes

Every CLI tool needs temporary files. They're like scratch paper for computers—essential but meant to be thrown away.

```javascript
import os from 'os';
import crypto from 'crypto';
import { promises as fs } from 'fs';

class TempFileManager {
  constructor() {
    this.tempFiles = new Set();
    this.tempDirs = new Set();

    // Clean up on exit
    process.on('exit', () => this.cleanup());
    process.on('SIGINT', () => {
      this.cleanup();
      process.exit(130);
    });
  }

  async createTempFile(options = {}) {
    const {
      prefix = 'tmp-',
      suffix = '',
      dir = os.tmpdir(),
      keep = false
    } = options;

    const randomName = crypto.randomBytes(16).toString('hex');
    const fileName = `${prefix}${randomName}${suffix}`;
    const filePath = path.join(dir, fileName);

    // Create the file
    const handle = await fs.open(filePath, 'w');
    await handle.close();

    if (!keep) {
      this.tempFiles.add(filePath);
    }

    return {
      path: filePath,
      cleanup: () => this.removeTempFile(filePath),
      write: (data) => fs.writeFile(filePath, data),
      read: () => fs.readFile(filePath, 'utf8'),
      append: (data) => fs.appendFile(filePath, data)
    };
  }

  async createTempDir(options = {}) {
    const {
      prefix = 'tmp-dir-',
      dir = os.tmpdir(),
      keep = false
    } = options;

    const randomName = crypto.randomBytes(16).toString('hex');
    const dirName = `${prefix}${randomName}`;
    const dirPath = path.join(dir, dirName);

    await fs.mkdir(dirPath, { recursive: true });

    if (!keep) {
      this.tempDirs.add(dirPath);
    }

    return {
      path: dirPath,
      cleanup: () => this.removeTempDir(dirPath),
      createFile: (name, content = '') => {
        const filePath = path.join(dirPath, name);
        return fs.writeFile(filePath, content);
      },
      listFiles: () => fs.readdir(dirPath)
    };
  }

  async withTempFile(fn, options = {}) {
    const temp = await this.createTempFile(options);

    try {
      return await fn(temp);
    } finally {
      await temp.cleanup();
    }
  }

  async withTempDir(fn, options = {}) {
    const temp = await this.createTempDir(options);

    try {
      return await fn(temp);
    } finally {
      await temp.cleanup();
    }
  }

  async removeTempFile(filePath) {
    try {
      await fs.unlink(filePath);
      this.tempFiles.delete(filePath);
    } catch (err) {
      // File might already be deleted
      if (err.code !== 'ENOENT') {
        console.error(`Failed to remove temp file: ${filePath}`);
      }
    }
  }

  async removeTempDir(dirPath) {
    try {
      await fs.rm(dirPath, { recursive: true, force: true });
      this.tempDirs.delete(dirPath);
    } catch (err) {
      console.error(`Failed to remove temp dir: ${dirPath}`);
    }
  }

  async cleanup() {
    // Clean up all tracked temp files and directories
    const cleanupPromises = [
      ...Array.from(this.tempFiles).map(f => this.removeTempFile(f)),
      ...Array.from(this.tempDirs).map(d => this.removeTempDir(d))
    ];

    await Promise.all(cleanupPromises);
  }
}

// Usage
const tempManager = new TempFileManager();

// Simple temp file
const temp = await tempManager.createTempFile({ suffix: '.json' });
await temp.write(JSON.stringify({ data: 'test' }));
const content = await temp.read();
await temp.cleanup();

// With automatic cleanup
await tempManager.withTempFile(async (temp) => {
  await temp.write('temporary data');
  const data = await temp.read();
  // File is automatically cleaned up when this function returns
});

// Temp directory for complex operations
await tempManager.withTempDir(async (temp) => {
  await temp.createFile('data.json', '{"test": true}');
  await temp.createFile('config.yml', 'key: value');
  const files = await temp.listFiles();
  console.log('Created files:', files);
  // Directory and all contents cleaned up automatically
});
```

## File Locking: Preventing Chaos

When multiple processes access the same files, chaos ensues. File locking prevents this chaos (mostly).

```javascript
import lockfile from 'proper-lockfile';

class FileLocker {
  constructor() {
    this.locks = new Map();
  }

  async withLock(filePath, fn, options = {}) {
    const {
      retries = 10,
      retryInterval = 100,
      stale = 10000  // Consider lock stale after 10 seconds
    } = options;

    let release;

    try {
      // Acquire lock
      release = await lockfile.lock(filePath, {
        retries: {
          retries,
          minTimeout: retryInterval,
          maxTimeout: retryInterval * 2
        },
        stale
      });

      this.locks.set(filePath, release);

      // Execute function with lock held
      return await fn();
    } finally {
      // Always release lock
      if (release) {
        await release();
        this.locks.delete(filePath);
      }
    }
  }

  async readWithLock(filePath) {
    return this.withLock(filePath, async () => {
      return fs.readFile(filePath, 'utf8');
    });
  }

  async writeWithLock(filePath, data) {
    return this.withLock(filePath, async () => {
      await fs.writeFile(filePath, data);
    });
  }

  async updateWithLock(filePath, updateFn) {
    return this.withLock(filePath, async () => {
      const current = await fs.readFile(filePath, 'utf8');
      const updated = await updateFn(current);
      await fs.writeFile(filePath, updated);
    });
  }
}

// Atomic file operations
class AtomicFileOperations {
  async writeAtomic(filePath, data) {
    const tempPath = `${filePath}.tmp.${Date.now()}`;

    try {
      // Write to temp file
      await fs.writeFile(tempPath, data);

      // Atomic rename
      await fs.rename(tempPath, filePath);
    } catch (err) {
      // Clean up temp file if something went wrong
      try {
        await fs.unlink(tempPath);
      } catch {
        // Ignore cleanup errors
      }
      throw err;
    }
  }

  async updateAtomic(filePath, updateFn) {
    const locker = new FileLocker();

    return locker.withLock(filePath, async () => {
      const current = await fs.readFile(filePath, 'utf8');
      const updated = await updateFn(current);
      await this.writeAtomic(filePath, updated);
    });
  }
}
```

## File System Utilities: The Swiss Army Knife

Every CLI tool eventually builds these utilities. Here they are, so you don't have to:

```javascript
class FileSystemUtils {
  async exists(path) {
    try {
      await fs.access(path);
      return true;
    } catch {
      return false;
    }
  }

  async isDirectory(path) {
    try {
      const stats = await fs.stat(path);
      return stats.isDirectory();
    } catch {
      return false;
    }
  }

  async isFile(path) {
    try {
      const stats = await fs.stat(path);
      return stats.isFile();
    } catch {
      return false;
    }
  }

  async getSize(path) {
    const stats = await fs.stat(path);

    if (stats.isFile()) {
      return stats.size;
    }

    if (stats.isDirectory()) {
      return this.getDirectorySize(path);
    }

    return 0;
  }

  async getDirectorySize(dirPath) {
    let totalSize = 0;
    const entries = await fs.readdir(dirPath, { withFileTypes: true });

    for (const entry of entries) {
      const fullPath = path.join(dirPath, entry.name);

      if (entry.isDirectory()) {
        totalSize += await this.getDirectorySize(fullPath);
      } else {
        const stats = await fs.stat(fullPath);
        totalSize += stats.size;
      }
    }

    return totalSize;
  }

  async findUpward(filename, startDir = process.cwd()) {
    let currentDir = path.resolve(startDir);

    while (true) {
      const filePath = path.join(currentDir, filename);

      if (await this.exists(filePath)) {
        return filePath;
      }

      const parentDir = path.dirname(currentDir);

      if (parentDir === currentDir) {
        // Reached root directory
        return null;
      }

      currentDir = parentDir;
    }
  }

  async tree(dirPath, options = {}) {
    const {
      maxDepth = Infinity,
      showHidden = false,
      showSize = false,
      currentDepth = 0,
      prefix = ''
    } = options;

    if (currentDepth >= maxDepth) {
      return '';
    }

    const entries = await fs.readdir(dirPath, { withFileTypes: true });
    const filtered = showHidden
      ? entries
      : entries.filter(e => !e.name.startsWith('.'));

    const sorted = filtered.sort((a, b) => {
      // Directories first, then alphabetical
      if (a.isDirectory() && !b.isDirectory()) return -1;
      if (!a.isDirectory() && b.isDirectory()) return 1;
      return a.name.localeCompare(b.name);
    });

    let output = '';

    for (let i = 0; i < sorted.length; i++) {
      const entry = sorted[i];
      const isLast = i === sorted.length - 1;
      const fullPath = path.join(dirPath, entry.name);

      const connector = isLast ? '└── ' : '├── ';
      const extension = isLast ? '    ' : '│   ';

      let line = prefix + connector;

      if (entry.isDirectory()) {
        line += chalk.blue(entry.name + '/');
      } else {
        line += entry.name;
      }

      if (showSize && entry.isFile()) {
        const stats = await fs.stat(fullPath);
        const size = this.formatFileSize(stats.size);
        line += chalk.gray(` (${size})`);
      }

      output += line + '\n';

      if (entry.isDirectory()) {
        output += await this.tree(fullPath, {
          ...options,
          currentDepth: currentDepth + 1,
          prefix: prefix + extension
        });
      }
    }

    return output;
  }

  formatFileSize(bytes) {
    const units = ['B', 'KB', 'MB', 'GB', 'TB'];
    let size = bytes;
    let unitIndex = 0;

    while (size >= 1024 && unitIndex < units.length - 1) {
      size /= 1024;
      unitIndex++;
    }

    return `${size.toFixed(1)}${units[unitIndex]}`;
  }

  async compareFiles(file1, file2) {
    const [stats1, stats2] = await Promise.all([
      fs.stat(file1),
      fs.stat(file2)
    ]);

    // Quick check: different sizes mean different files
    if (stats1.size !== stats2.size) {
      return false;
    }

    // Compare content
    const [content1, content2] = await Promise.all([
      fs.readFile(file1),
      fs.readFile(file2)
    ]);

    return content1.equals(content2);
  }
}
```

## Real-World Example: A File Sync Tool

Let's put it all together with a real-world example—a file synchronization tool:

```javascript
import crypto from 'crypto';

class FileSyncTool {
  constructor() {
    this.utils = new FileSystemUtils();
    this.watcher = new FileWatcher();
    this.checksums = new Map();
  }

  async sync(source, destination, options = {}) {
    const {
      watch = false,
      delete: deleteExtra = false,
      exclude = [],
      dryRun = false,
      verbose = false
    } = options;

    console.log(chalk.bold('\n🔄 File Synchronization Tool\n'));
    console.log(chalk.gray(`Source:      ${source}`));
    console.log(chalk.gray(`Destination: ${destination}`));
    console.log(chalk.gray(`Mode:        ${dryRun ? 'Dry run' : 'Live'}\n`));

    // Initial sync
    await this.performSync(source, destination, options);

    if (watch && !dryRun) {
      console.log(chalk.cyan('\n👁️  Watching for changes...\n'));

      this.watcher.watch(source, {
        ignored: exclude
      }).on('all', async (event, filePath) => {
        const relativePath = path.relative(source, filePath);
        const destPath = path.join(destination, relativePath);

        if (verbose) {
          console.log(chalk.gray(`[${new Date().toISOString()}] ${event}: ${relativePath}`));
        }

        switch (event) {
          case 'add':
          case 'change':
            await this.syncFile(filePath, destPath, options);
            break;
          case 'unlink':
            if (deleteExtra && !dryRun) {
              await this.utils.remove(destPath);
              console.log(chalk.red(`✖ Deleted: ${relativePath}`));
            }
            break;
        }
      });
    }
  }

  async performSync(source, destination, options) {
    const spinner = ora('Analyzing files...').start();

    // Get all files in source
    const sourceFiles = await this.getAllFiles(source, options.exclude);

    // Get all files in destination
    const destFiles = await this.getAllFiles(destination, options.exclude);

    spinner.text = 'Comparing files...';

    // Determine what needs to be synced
    const toCreate = [];
    const toUpdate = [];
    const toDelete = [];

    for (const file of sourceFiles) {
      const relativePath = path.relative(source, file);
      const destPath = path.join(destination, relativePath);

      if (!await this.utils.exists(destPath)) {
        toCreate.push({ source: file, dest: destPath, relative: relativePath });
      } else if (await this.needsUpdate(file, destPath)) {
        toUpdate.push({ source: file, dest: destPath, relative: relativePath });
      }
    }

    if (options.delete) {
      for (const file of destFiles) {
        const relativePath = path.relative(destination, file);
        const sourcePath = path.join(source, relativePath);

        if (!await this.utils.exists(sourcePath)) {
          toDelete.push({ path: file, relative: relativePath });
        }
      }
    }

    spinner.stop();

    // Show summary
    console.log(chalk.bold('📋 Sync Summary:\n'));
    console.log(chalk.green(`  ✚ Create: ${toCreate.length} files`));
    console.log(chalk.yellow(`  ✎ Update: ${toUpdate.length} files`));
    console.log(chalk.red(`  ✖ Delete: ${toDelete.length} files`));

    if (toCreate.length === 0 && toUpdate.length === 0 && toDelete.length === 0) {
      console.log(chalk.gray('\n✓ Everything is in sync!'));
      return;
    }

    if (options.dryRun) {
      console.log(chalk.yellow('\n⚠ Dry run mode - no changes made'));
      return;
    }

    // Confirm before proceeding
    const { proceed } = await inquirer.prompt([
      {
        type: 'confirm',
        name: 'proceed',
        message: 'Proceed with synchronization?',
        default: true
      }
    ]);

    if (!proceed) {
      console.log(chalk.yellow('Synchronization cancelled'));
      return;
    }

    // Perform sync
    const progress = ora('Synchronizing files...').start();

    for (const { source, dest, relative } of toCreate) {
      await this.syncFile(source, dest, options);
      if (options.verbose) {
        progress.text = `Created: ${relative}`;
      }
    }

    for (const { source, dest, relative } of toUpdate) {
      await this.syncFile(source, dest, options);
      if (options.verbose) {
        progress.text = `Updated: ${relative}`;
      }
    }

    for (const { path, relative } of toDelete) {
      await this.utils.remove(path);
      if (options.verbose) {
        progress.text = `Deleted: ${relative}`;
      }
    }

    progress.succeed('Synchronization complete!');
  }

  async syncFile(source, destination, options) {
    // Ensure destination directory exists
    await this.utils.ensureDir(path.dirname(destination));

    // Copy file
    await fs.copyFile(source, destination);

    // Preserve timestamps if needed
    if (options.preserveTimestamps) {
      const stats = await fs.stat(source);
      await fs.utimes(destination, stats.atime, stats.mtime);
    }
  }

  async needsUpdate(source, destination) {
    const [sourceStats, destStats] = await Promise.all([
      fs.stat(source),
      fs.stat(destination)
    ]);

    // Check modification time
    if (sourceStats.mtime > destStats.mtime) {
      return true;
    }

    // Check size
    if (sourceStats.size !== destStats.size) {
      return true;
    }

    // Check content (expensive, so do last)
    const [sourceHash, destHash] = await Promise.all([
      this.getFileHash(source),
      this.getFileHash(destination)
    ]);

    return sourceHash !== destHash;
  }

  async getFileHash(filePath) {
    // Check cache
    if (this.checksums.has(filePath)) {
      return this.checksums.get(filePath);
    }

    const hash = crypto.createHash('sha256');
    const stream = createReadStream(filePath);

    return new Promise((resolve, reject) => {
      stream.on('data', chunk => hash.update(chunk));
      stream.on('end', () => {
        const checksum = hash.digest('hex');
        this.checksums.set(filePath, checksum);
        resolve(checksum);
      });
      stream.on('error', reject);
    });
  }

  async getAllFiles(dir, exclude = []) {
    const files = [];

    async function walk(currentPath) {
      const entries = await fs.readdir(currentPath, { withFileTypes: true });

      for (const entry of entries) {
        const fullPath = path.join(currentPath, entry.name);
        const relativePath = path.relative(dir, fullPath);

        // Check if excluded
        if (exclude.some(pattern => micromatch.isMatch(relativePath, pattern))) {
          continue;
        }

        if (entry.isDirectory()) {
          await walk(fullPath);
        } else {
          files.push(fullPath);
        }
      }
    }

    await walk(dir);
    return files;
  }
}

// Usage
const syncer = new FileSyncTool();
await syncer.sync('./source', './backup', {
  watch: true,
  delete: true,
  exclude: ['*.log', 'node_modules/**'],
  verbose: true
});
```

## Lessons from the Filesystem

Working with files in Node.js teaches you humility. Just when you think you've handled every edge case, someone runs your tool on a case-insensitive filesystem, or a network drive, or a directory with 10,000 files named with emojis.

Here are the hard-won lessons:

1. **Always handle errors**: Files disappear, permissions change, disks fill up. Your tool needs to handle it all gracefully.

2. **Stream when possible**: Loading a 2GB file into memory is a great way to crash your tool and annoy your users.

3. **Respect the platform**: Windows paths are different. Line endings are different. File permissions are different. Deal with it.

4. **Clean up after yourself**: Temporary files that aren't temporary are just litter. Clean them up.

5. **Watch performance**: File I/O is slow. Batch operations, use streams, cache results.

6. **Provide feedback**: File operations can take time. Show progress, explain what's happening.

7. **Be atomic**: Half-written files are worse than no files. Write to temp, then rename.

The filesystem is where your CLI tool proves its worth. It's not glamorous work—nobody gets excited about path normalization or file locking—but it's essential. Every tool that matters spends most of its time reading, writing, watching, and manipulating files.

Master the filesystem, and you've mastered 80% of CLI development. The other 20% is just making it look pretty while it happens.