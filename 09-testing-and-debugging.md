# Chapter 9: Testing and Debugging CLI Applications

Testing CLI tools is like debugging a conversation between your code, the operating system, the terminal, and a human who might be running your tool at 3 AM while three cups of coffee deep into a production emergency. It's complex, it's necessary, and it's weirdly satisfying when you finally catch that bug that only manifests on Windows when the username contains unicode characters.

The challenge of testing CLI tools is that they interact with everything: the filesystem, environment variables, stdin/stdout, child processes, and the user's patience. Unlike a pure function that takes an input and returns an output, CLI tools have side effects galore. They read files, write files, delete files, start servers, kill processes, and occasionally achieve sentience (usually around version 2.0).

## The Testing Philosophy

Before we dive into code, let's establish a philosophy. CLI tool testing has three levels:

1. **Unit Tests**: Test individual functions in isolation
2. **Integration Tests**: Test components working together
3. **End-to-End Tests**: Test the actual CLI commands as users would run them

The pyramid principle applies: lots of unit tests, some integration tests, few but critical end-to-end tests. CLI tools have a tendency to invert this pyramid because "it's easier to just test the whole thing." Resist this urge. Your future self debugging a failed CI pipeline at midnight will thank you.

```javascript
// Good test structure
// 80% Unit tests    ████████████████████
// 15% Integration   ███
// 5%  E2E          █

// What usually happens
// 20% Unit tests    ████
// 20% Integration   ████
// 60% E2E          ████████████

// The E2E tests fail, you don't know why, and it's 2 AM
```

## Unit Testing: The Foundation

Unit tests for CLI tools test the logic, not the interface:

```javascript
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import { vi } from 'vitest';
import fs from 'fs/promises';
import path from 'path';
import os from 'os';

// Code under test
class FileProcessor {
  constructor(options = {}) {
    this.encoding = options.encoding || 'utf8';
    this.maxFileSize = options.maxFileSize || 10 * 1024 * 1024; // 10MB
  }

  async processFile(filepath) {
    const stats = await fs.stat(filepath);

    if (stats.size > this.maxFileSize) {
      throw new Error(`File too large: ${stats.size} bytes`);
    }

    const content = await fs.readFile(filepath, this.encoding);
    return this.transform(content);
  }

  transform(content) {
    // Business logic
    return content
      .split('\n')
      .map(line => line.trim())
      .filter(line => line.length > 0)
      .map(line => `// ${line}`)
      .join('\n');
  }

  async processDirectory(dirpath) {
    const entries = await fs.readdir(dirpath, { withFileTypes: true });

    const results = [];
    for (const entry of entries) {
      if (entry.isFile() && entry.name.endsWith('.txt')) {
        const filepath = path.join(dirpath, entry.name);
        const processed = await this.processFile(filepath);
        results.push({ file: entry.name, content: processed });
      }
    }

    return results;
  }
}

// Unit tests
describe('FileProcessor', () => {
  let processor;

  beforeEach(() => {
    processor = new FileProcessor();
  });

  describe('transform', () => {
    it('should add comments to non-empty lines', () => {
      const input = 'line 1\n  line 2  \n\nline 3\n';
      const expected = '// line 1\n// line 2\n// line 3';

      expect(processor.transform(input)).toBe(expected);
    });

    it('should handle empty input', () => {
      expect(processor.transform('')).toBe('');
    });

    it('should handle whitespace-only input', () => {
      expect(processor.transform('   \n  \n \n')).toBe('');
    });
  });

  describe('processFile', () => {
    let tempDir;
    let testFile;

    beforeEach(async () => {
      tempDir = await fs.mkdtemp(path.join(os.tmpdir(), 'test-'));
      testFile = path.join(tempDir, 'test.txt');
    });

    afterEach(async () => {
      await fs.rm(tempDir, { recursive: true, force: true });
    });

    it('should process a valid file', async () => {
      await fs.writeFile(testFile, 'hello\nworld\n');

      const result = await processor.processFile(testFile);

      expect(result).toBe('// hello\n// world');
    });

    it('should reject files that are too large', async () => {
      processor.maxFileSize = 10; // 10 bytes
      await fs.writeFile(testFile, 'this is a very long file content');

      await expect(processor.processFile(testFile))
        .rejects
        .toThrow('File too large');
    });

    it('should handle non-existent files', async () => {
      await expect(processor.processFile('/does/not/exist'))
        .rejects
        .toThrow('ENOENT');
    });
  });

  describe('processDirectory', () => {
    let tempDir;

    beforeEach(async () => {
      tempDir = await fs.mkdtemp(path.join(os.tmpdir(), 'test-'));
    });

    afterEach(async () => {
      await fs.rm(tempDir, { recursive: true, force: true });
    });

    it('should process only .txt files', async () => {
      // Create test files
      await fs.writeFile(path.join(tempDir, 'file1.txt'), 'content 1');
      await fs.writeFile(path.join(tempDir, 'file2.txt'), 'content 2');
      await fs.writeFile(path.join(tempDir, 'file3.js'), 'ignored');
      await fs.mkdir(path.join(tempDir, 'subdir'));

      const results = await processor.processDirectory(tempDir);

      expect(results).toHaveLength(2);
      expect(results.map(r => r.file).sort()).toEqual(['file1.txt', 'file2.txt']);
      expect(results[0].content).toBe('// content 1');
    });

    it('should handle empty directory', async () => {
      const results = await processor.processDirectory(tempDir);
      expect(results).toEqual([]);
    });
  });
});
```

## Mocking: Faking the World

CLI tools interact with the world. In tests, we need to fake the world:

```javascript
import { vi } from 'vitest';

class CLITestHelpers {
  constructor() {
    this.mocks = {
      fs: vi.hoisted(() => ({
        readFile: vi.fn(),
        writeFile: vi.fn(),
        stat: vi.fn(),
        readdir: vi.fn(),
        mkdir: vi.fn(),
        rm: vi.fn()
      })),
      process: {
        stdout: {
          write: vi.fn(),
          columns: 80,
          rows: 24
        },
        stderr: {
          write: vi.fn()
        },
        exit: vi.fn(),
        cwd: vi.fn(() => '/test/cwd'),
        env: { ...process.env, NODE_ENV: 'test' }
      },
      exec: vi.fn()
    };
  }

  mockFileSystem(files = {}) {
    // Mock fs operations based on a virtual file structure
    this.mocks.fs.readFile.mockImplementation(async (filepath, encoding) => {
      if (files[filepath]) {
        return encoding ? files[filepath] : Buffer.from(files[filepath]);
      }
      throw Object.assign(new Error(`ENOENT: no such file or directory, open '${filepath}'`), {
        code: 'ENOENT',
        path: filepath
      });
    });

    this.mocks.fs.stat.mockImplementation(async (filepath) => {
      if (files[filepath]) {
        return {
          isFile: () => true,
          isDirectory: () => false,
          size: files[filepath].length,
          mtime: new Date()
        };
      }
      throw Object.assign(new Error(`ENOENT: no such file or directory, stat '${filepath}'`), {
        code: 'ENOENT',
        path: filepath
      });
    });

    this.mocks.fs.writeFile.mockImplementation(async (filepath, content) => {
      files[filepath] = content;
    });

    this.mocks.fs.readdir.mockImplementation(async (dirpath) => {
      const entries = Object.keys(files)
        .filter(f => f.startsWith(dirpath))
        .map(f => f.substring(dirpath.length + 1))
        .filter(f => !f.includes('/'));

      return entries.map(name => ({
        name,
        isFile: () => true,
        isDirectory: () => false
      }));
    });
  }

  mockCommand(command, result) {
    this.mocks.exec.mockImplementation((cmd) => {
      if (cmd.includes(command)) {
        if (result instanceof Error) {
          return Promise.reject(result);
        }
        return Promise.resolve({
          stdout: result.stdout || '',
          stderr: result.stderr || ''
        });
      }
      throw new Error(`Unexpected command: ${cmd}`);
    });
  }

  captureOutput() {
    const stdout = [];
    const stderr = [];

    this.mocks.process.stdout.write.mockImplementation((data) => {
      stdout.push(data);
    });

    this.mocks.process.stderr.write.mockImplementation((data) => {
      stderr.push(data);
    });

    return {
      getStdout: () => stdout.join(''),
      getStderr: () => stderr.join(''),
      clear: () => {
        stdout.length = 0;
        stderr.length = 0;
      }
    };
  }

  setupTestEnvironment() {
    // Mock global modules
    vi.mock('fs/promises', () => this.mocks.fs);
    vi.mock('child_process', () => ({ exec: this.mocks.exec }));

    // Mock process
    Object.assign(process, this.mocks.process);

    return this;
  }

  cleanup() {
    vi.restoreAllMocks();
  }
}

// Using the test helpers
describe('Git CLI Tool', () => {
  let helpers;
  let output;

  beforeEach(() => {
    helpers = new CLITestHelpers().setupTestEnvironment();
    output = helpers.captureOutput();
  });

  afterEach(() => {
    helpers.cleanup();
  });

  it('should show git status', async () => {
    helpers.mockCommand('git status', {
      stdout: 'On branch main\nnothing to commit, working tree clean'
    });

    await runGitCommand(['status']);

    expect(output.getStdout()).toContain('working tree clean');
  });

  it('should handle git errors gracefully', async () => {
    helpers.mockCommand('git status', new Error('Not a git repository'));

    await runGitCommand(['status']);

    expect(output.getStderr()).toContain('Not a git repository');
    expect(helpers.mocks.process.exit).toHaveBeenCalledWith(1);
  });
});
```

## Integration Testing: Components Together

Integration tests verify that different parts of your CLI work together:

```javascript
import { spawn } from 'child_process';
import { promisify } from 'util';
import fs from 'fs/promises';
import path from 'path';
import os from 'os';

class CLIIntegrationTest {
  constructor(cliPath) {
    this.cliPath = cliPath;
    this.workingDir = null;
  }

  async setup() {
    this.workingDir = await fs.mkdtemp(path.join(os.tmpdir(), 'cli-test-'));
    process.chdir(this.workingDir);
  }

  async cleanup() {
    if (this.workingDir) {
      await fs.rm(this.workingDir, { recursive: true, force: true });
    }
  }

  async run(args = [], options = {}) {
    return new Promise((resolve, reject) => {
      const {
        input = '',
        timeout = 10000,
        env = {}
      } = options;

      const child = spawn('node', [this.cliPath, ...args], {
        cwd: this.workingDir,
        env: { ...process.env, ...env },
        stdio: ['pipe', 'pipe', 'pipe']
      });

      let stdout = '';
      let stderr = '';

      child.stdout.on('data', (data) => {
        stdout += data.toString();
      });

      child.stderr.on('data', (data) => {
        stderr += data.toString();
      });

      const timer = setTimeout(() => {
        child.kill('SIGTERM');
        reject(new Error('Command timeout'));
      }, timeout);

      child.on('close', (code, signal) => {
        clearTimeout(timer);

        resolve({
          code,
          signal,
          stdout,
          stderr,
          success: code === 0
        });
      });

      child.on('error', (error) => {
        clearTimeout(timer);
        reject(error);
      });

      // Send input if provided
      if (input) {
        child.stdin.write(input);
      }
      child.stdin.end();
    });
  }

  async createFile(filepath, content) {
    const fullPath = path.join(this.workingDir, filepath);
    await fs.mkdir(path.dirname(fullPath), { recursive: true });
    await fs.writeFile(fullPath, content);
  }

  async readFile(filepath) {
    const fullPath = path.join(this.workingDir, filepath);
    return fs.readFile(fullPath, 'utf8');
  }

  async fileExists(filepath) {
    try {
      const fullPath = path.join(this.workingDir, filepath);
      await fs.access(fullPath);
      return true;
    } catch {
      return false;
    }
  }
}

describe('CLI Integration Tests', () => {
  let cli;

  beforeEach(async () => {
    cli = new CLIIntegrationTest('./dist/cli.js');
    await cli.setup();
  });

  afterEach(async () => {
    await cli.cleanup();
  });

  it('should process files correctly', async () => {
    // Setup test files
    await cli.createFile('input.txt', 'line 1\nline 2\n  line 3  \n');
    await cli.createFile('config.json', '{"encoding": "utf8"}');

    // Run CLI command
    const result = await cli.run(['process', 'input.txt', '--config', 'config.json']);

    // Verify results
    expect(result.success).toBe(true);
    expect(result.stdout).toContain('Processed 1 file');

    // Check output file
    const output = await cli.readFile('input.processed.txt');
    expect(output).toBe('// line 1\n// line 2\n// line 3');
  });

  it('should handle missing files gracefully', async () => {
    const result = await cli.run(['process', 'nonexistent.txt']);

    expect(result.success).toBe(false);
    expect(result.stderr).toContain('File not found');
    expect(result.code).toBe(1);
  });

  it('should support interactive mode', async () => {
    const result = await cli.run(['interactive'], {
      input: 'yes\ninput.txt\nexit\n'
    });

    expect(result.success).toBe(true);
    expect(result.stdout).toContain('Enter filename:');
  });

  it('should respect environment variables', async () => {
    await cli.createFile('input.txt', 'content');

    const result = await cli.run(['process', 'input.txt'], {
      env: { LOG_LEVEL: 'debug' }
    });

    expect(result.stdout).toContain('DEBUG:');
  });
});
```

## End-to-End Testing: The Full Experience

E2E tests verify the complete user experience:

```javascript
import { test, expect } from '@playwright/test';
import { execSync } from 'child_process';
import fs from 'fs/promises';

class E2ETestRunner {
  constructor() {
    this.testDir = null;
    this.originalDir = process.cwd();
  }

  async setupTestEnvironment() {
    this.testDir = await fs.mkdtemp('/tmp/e2e-test-');
    process.chdir(this.testDir);

    // Install the CLI tool
    execSync('npm install -g ../dist/my-cli-tool.tgz', { stdio: 'inherit' });
  }

  async cleanup() {
    process.chdir(this.originalDir);
    if (this.testDir) {
      await fs.rm(this.testDir, { recursive: true, force: true });
    }
    // Uninstall CLI tool
    try {
      execSync('npm uninstall -g my-cli-tool', { stdio: 'pipe' });
    } catch {
      // Ignore uninstall errors
    }
  }

  async runCLI(command) {
    return new Promise((resolve, reject) => {
      const proc = exec(command, (error, stdout, stderr) => {
        resolve({
          error,
          stdout,
          stderr,
          success: !error
        });
      });

      // Set a timeout
      setTimeout(() => {
        proc.kill();
        reject(new Error('E2E test timeout'));
      }, 30000);
    });
  }
}

describe('E2E Tests', () => {
  let runner;

  beforeAll(async () => {
    runner = new E2ETestRunner();
    await runner.setupTestEnvironment();
  });

  afterAll(async () => {
    await runner.cleanup();
  });

  test('complete workflow', async () => {
    // 1. Initialize a new project
    let result = await runner.runCLI('my-cli init my-project');
    expect(result.success).toBe(true);
    expect(result.stdout).toContain('Project created');

    // 2. Verify project structure
    const files = await fs.readdir('./my-project');
    expect(files).toContain('package.json');
    expect(files).toContain('src');

    // 3. Build the project
    process.chdir('./my-project');
    result = await runner.runCLI('my-cli build');
    expect(result.success).toBe(true);

    // 4. Verify build output
    const distFiles = await fs.readdir('./dist');
    expect(distFiles.length).toBeGreaterThan(0);

    // 5. Run tests
    result = await runner.runCLI('my-cli test');
    expect(result.success).toBe(true);
    expect(result.stdout).toContain('All tests passed');
  });

  test('error handling', async () => {
    const result = await runner.runCLI('my-cli build --invalid-flag');
    expect(result.success).toBe(false);
    expect(result.stderr).toContain('Unknown flag');
  });
});
```

## Visual Testing: Screenshots and Output

CLI tools have visual output. Test it:

```javascript
import stripAnsi from 'strip-ansi';
import chalk from 'chalk';

class VisualTester {
  constructor() {
    this.snapshots = new Map();
  }

  captureOutput(fn) {
    const originalWrite = process.stdout.write;
    const originalColumns = process.stdout.columns;
    let output = '';

    // Mock terminal size
    process.stdout.columns = 80;

    // Capture output
    process.stdout.write = (data) => {
      output += data;
      return true;
    };

    try {
      fn();
      return output;
    } finally {
      process.stdout.write = originalWrite;
      process.stdout.columns = originalColumns;
    }
  }

  normalizeOutput(output) {
    return stripAnsi(output)
      .replace(/\r\n/g, '\n')
      .replace(/\r/g, '\n')
      .trim();
  }

  matchSnapshot(name, output) {
    const normalized = this.normalizeOutput(output);
    const existing = this.snapshots.get(name);

    if (!existing) {
      this.snapshots.set(name, normalized);
      return { matches: true, isNew: true };
    }

    const matches = existing === normalized;
    return {
      matches,
      isNew: false,
      diff: matches ? null : this.generateDiff(existing, normalized)
    };
  }

  generateDiff(expected, actual) {
    const expectedLines = expected.split('\n');
    const actualLines = actual.split('\n');
    const maxLines = Math.max(expectedLines.length, actualLines.length);

    const diff = [];
    for (let i = 0; i < maxLines; i++) {
      const expectedLine = expectedLines[i] || '';
      const actualLine = actualLines[i] || '';

      if (expectedLine !== actualLine) {
        diff.push(`Line ${i + 1}:`);
        diff.push(`- ${expectedLine}`);
        diff.push(`+ ${actualLine}`);
      }
    }

    return diff.join('\n');
  }
}

describe('Visual Output Tests', () => {
  let tester;

  beforeEach(() => {
    tester = new VisualTester();
  });

  it('should render help text correctly', () => {
    const output = tester.captureOutput(() => {
      console.log(chalk.bold('My CLI Tool v1.0.0'));
      console.log(chalk.gray('Usage: my-cli [command] [options]'));
      console.log();
      console.log(chalk.yellow('Commands:'));
      console.log('  init     Initialize project');
      console.log('  build    Build project');
      console.log('  test     Run tests');
    });

    const result = tester.matchSnapshot('help-text', output);
    expect(result.matches).toBe(true);
  });

  it('should render progress bars correctly', () => {
    const output = tester.captureOutput(() => {
      // Simulate progress bar
      for (let i = 0; i <= 10; i++) {
        const percent = (i / 10) * 100;
        const filled = '█'.repeat(Math.floor(i / 2));
        const empty = '░'.repeat(5 - Math.floor(i / 2));
        process.stdout.write(`\r[${filled}${empty}] ${percent.toFixed(0)}%`);
      }
      console.log('\nComplete!');
    });

    const result = tester.matchSnapshot('progress-bar', output);
    expect(result.matches).toBe(true);
  });
});
```

## Error Testing: When Things Go Wrong

Testing error conditions is crucial for CLI tools:

```javascript
describe('Error Handling', () => {
  it('should handle filesystem errors', async () => {
    const mockFs = {
      readFile: vi.fn().mockRejectedValue(
        Object.assign(new Error('Permission denied'), {
          code: 'EACCES',
          path: '/protected/file.txt'
        })
      )
    };

    vi.mock('fs/promises', () => mockFs);

    const cli = new MyCLI();
    const result = await cli.processFile('/protected/file.txt');

    expect(result.success).toBe(false);
    expect(result.error.code).toBe('EACCES');
    expect(result.message).toContain('Permission denied');
  });

  it('should handle network timeouts', async () => {
    const mockFetch = vi.fn().mockImplementation(() => {
      return new Promise((_, reject) => {
        setTimeout(() => reject(new Error('Network timeout')), 100);
      });
    });

    global.fetch = mockFetch;

    const cli = new MyCLI();
    const result = await cli.fetchData('https://api.example.com/data');

    expect(result.success).toBe(false);
    expect(result.error.message).toContain('timeout');
  });

  it('should handle invalid JSON gracefully', async () => {
    const cli = new MyCLI();

    await expect(cli.parseConfig('{ invalid json }')).rejects.toThrow('Invalid JSON');
  });

  it('should validate user input', async () => {
    const cli = new MyCLI();

    await expect(cli.createProject('')).rejects.toThrow('Project name required');
    await expect(cli.createProject('invalid/name')).rejects.toThrow('Invalid project name');
    await expect(cli.createProject('.hidden')).rejects.toThrow('Project name cannot start with dot');
  });
});
```

## Debugging: When Tests Fail

When tests fail (and they will), you need debugging strategies:

```javascript
class CLIDebugger {
  constructor() {
    this.debugMode = process.env.DEBUG === 'true';
    this.logLevel = process.env.LOG_LEVEL || 'info';
  }

  log(level, message, data = null) {
    if (!this.debugMode && level === 'debug') return;

    const timestamp = new Date().toISOString();
    const prefix = `[${timestamp}] ${level.toUpperCase()}:`;

    console.log(prefix, message);

    if (data) {
      console.log(JSON.stringify(data, null, 2));
    }
  }

  debug(message, data) {
    this.log('debug', message, data);
  }

  info(message, data) {
    this.log('info', message, data);
  }

  error(message, error) {
    this.log('error', message, {
      message: error.message,
      stack: error.stack,
      code: error.code
    });
  }

  // Create a debugging wrapper for any function
  trace(fn, name = fn.name) {
    return async (...args) => {
      const start = Date.now();
      this.debug(`Starting ${name}`, { args });

      try {
        const result = await fn(...args);
        const duration = Date.now() - start;
        this.debug(`Completed ${name}`, { duration, result });
        return result;
      } catch (error) {
        const duration = Date.now() - start;
        this.error(`Failed ${name}`, error);
        this.debug(`Failed ${name}`, { duration, args });
        throw error;
      }
    };
  }

  // Memory and performance debugging
  snapshot(label = 'snapshot') {
    if (!this.debugMode) return;

    const usage = process.memoryUsage();
    const uptime = process.uptime();

    this.debug(`Memory snapshot: ${label}`, {
      rss: `${(usage.rss / 1024 / 1024).toFixed(2)}MB`,
      heapUsed: `${(usage.heapUsed / 1024 / 1024).toFixed(2)}MB`,
      heapTotal: `${(usage.heapTotal / 1024 / 1024).toFixed(2)}MB`,
      external: `${(usage.external / 1024 / 1024).toFixed(2)}MB`,
      uptime: `${uptime.toFixed(2)}s`
    });
  }

  // Create a test recorder for reproducing issues
  startRecording() {
    const events = [];

    const originalWrite = process.stdout.write;
    const originalExit = process.exit;

    process.stdout.write = (data) => {
      events.push({ type: 'stdout', data: data.toString(), timestamp: Date.now() });
      return originalWrite.call(process.stdout, data);
    };

    process.exit = (code) => {
      events.push({ type: 'exit', code, timestamp: Date.now() });
      return originalExit.call(process, code);
    };

    return {
      stop: () => {
        process.stdout.write = originalWrite;
        process.exit = originalExit;
        return events;
      },

      save: (filename) => {
        const recording = {
          events,
          metadata: {
            platform: process.platform,
            nodeVersion: process.version,
            argv: process.argv,
            env: process.env,
            cwd: process.cwd()
          }
        };

        fs.writeFileSync(filename, JSON.stringify(recording, null, 2));
      }
    };
  }
}

// Usage in tests
describe('CLI with debugging', () => {
  let debugger;

  beforeEach(() => {
    debugger = new CLIDebugger();
  });

  it('should process files with debug info', async () => {
    const recorder = debugger.startRecording();

    try {
      const cli = new MyCLI();
      debugger.snapshot('before-processing');

      const processFile = debugger.trace(cli.processFile.bind(cli), 'processFile');
      await processFile('test.txt');

      debugger.snapshot('after-processing');
    } catch (error) {
      const events = recorder.stop();
      recorder.save('failed-test-recording.json');
      throw error;
    }
  });
});
```

## Test Organization and Best Practices

Here's how to structure your test suite:

```javascript
// tests/
//   ├── unit/
//   │   ├── file-processor.test.js
//   │   ├── config-parser.test.js
//   │   └── utils.test.js
//   ├── integration/
//   │   ├── cli-commands.test.js
//   │   └── workflow.test.js
//   ├── e2e/
//   │   └── complete-workflow.test.js
//   ├── fixtures/
//   │   ├── sample-config.json
//   │   └── test-files/
//   └── helpers/
//       ├── test-runner.js
//       └── mocks.js

// vitest.config.js
export default {
  test: {
    globals: true,
    environment: 'node',
    setupFiles: ['./tests/setup.js'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html'],
      exclude: [
        'tests/**',
        'dist/**',
        '**/*.config.js'
      ]
    },
    testMatch: [
      '**/tests/**/*.test.js'
    ],
    testTimeout: 10000
  }
};

// tests/setup.js
import { beforeAll, afterAll } from 'vitest';

beforeAll(() => {
  // Global test setup
  process.env.NODE_ENV = 'test';
  process.env.LOG_LEVEL = 'error';
});

afterAll(() => {
  // Global cleanup
});

// Test patterns to follow
class TestPatterns {
  // Arrange, Act, Assert pattern
  async testAAA() {
    // Arrange - set up test data
    const input = 'test input';
    const expectedOutput = 'expected result';

    // Act - perform the action
    const result = await functionUnderTest(input);

    // Assert - verify the result
    expect(result).toBe(expectedOutput);
  }

  // Given, When, Then pattern
  async testGWT() {
    // Given this state
    const cli = new MyCLI({ debug: true });
    await cli.createFile('test.txt', 'content');

    // When this action occurs
    const result = await cli.processFile('test.txt');

    // Then this should happen
    expect(result.success).toBe(true);
    expect(result.output).toContain('processed');
  }

  // Test naming convention
  // [Method]_Should[Expected]_When[Condition]
  async processFile_ShouldThrowError_WhenFileNotFound() {
    const cli = new MyCLI();

    await expect(cli.processFile('nonexistent.txt'))
      .rejects
      .toThrow('File not found');
  }
}
```

## Continuous Integration: Automated Testing

Set up CI/CD for your CLI tool:

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: [16, 18, 20]

    steps:
    - uses: actions/checkout@v3

    - name: Use Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v3
      with:
        node-version: ${{ matrix.node-version }}

    - name: Install dependencies
      run: npm ci

    - name: Run linting
      run: npm run lint

    - name: Run unit tests
      run: npm run test:unit

    - name: Run integration tests
      run: npm run test:integration

    - name: Run E2E tests
      run: npm run test:e2e

    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage/lcov.info

  test-cli:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3

    - name: Build CLI
      run: npm run build

    - name: Test CLI installation
      run: |
        npm pack
        npm install -g *.tgz
        my-cli --version
        my-cli --help

    - name: Test CLI functionality
      run: |
        mkdir test-project
        cd test-project
        my-cli init
        my-cli build
        my-cli test
```

## The Testing Mindset

Testing CLI tools teaches you patience and humility. Your code will break in ways you never imagined. Users will run your tool on operating systems you've never heard of, with terminal emulators that interpret ANSI codes as suggestions rather than commands.

The key principles:

1. **Test the happy path first**: Make sure basic functionality works
2. **Then test the edge cases**: Empty files, huge files, Unicode, Windows
3. **Test error conditions**: What happens when things go wrong?
4. **Test user interactions**: Mock stdin/stdout carefully
5. **Test across platforms**: Windows is different. Accept it.

Remember: The goal isn't 100% test coverage. The goal is confidence. Confidence that your tool works, confidence that changes don't break existing functionality, and confidence that when something does break, you'll find out quickly.

In the final chapter, we'll explore distribution and updates—how to get your carefully tested CLI tool into the hands of users, and how to keep it updated without breaking their workflows. Because a tool that works perfectly in your tests but crashes on the user's machine is just an expensive unit test.