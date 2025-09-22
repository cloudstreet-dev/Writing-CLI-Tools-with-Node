# Chapter 3: Building Your First CLI Tool

Let's build something. Not another todo app (the world has enough of those), not a weather checker (boring), and definitely not another "hello world" (we're better than that). We're going to build a CLI tool that every developer secretly wants: a commit message generator that sounds intelligent even when you've just spent three hours renaming variables.

But first, let's talk about the moment every CLI tool is born: the shebang.

## The Sacred Shebang

```javascript
#!/usr/bin/env node

console.log("I'm a real CLI tool now!");
```

That first line—the shebang—is the magical incantation that transforms a JavaScript file into a command. It tells the system, "Hey, when someone runs this file, use Node.js to interpret it." Without it, you're just another JavaScript file. With it, you're a CLI tool. It's like the difference between Clark Kent and Superman, if Superman's superpower was parsing command-line arguments.

The `env` part is important. You could write `#!/usr/bin/node`, but that assumes Node.js lives at `/usr/bin/node`. Using `env` lets the system find Node.js wherever it's hiding, which is especially important when dealing with version managers like nvm, where Node.js changes locations more often than a witness in protection.

## Project Setup: The Boring but Necessary Part

Every CLI tool starts with the same ritual:

```bash
mkdir commit-genius
cd commit-genius
npm init -y
```

Now comes the important part—editing `package.json` to make your script executable:

```json
{
  "name": "commit-genius",
  "version": "1.0.0",
  "description": "Generate commit messages that make you look smarter",
  "main": "index.js",
  "type": "module",
  "bin": {
    "commit-genius": "./index.js"
  },
  "scripts": {
    "test": "echo \"Tests? Where we're going, we don't need tests.\" && exit 1"
  },
  "keywords": ["cli", "commit", "artificial-intelligence", "imposter-syndrome"],
  "author": "Someone Who Comments Their Code",
  "license": "MIT"
}
```

The `bin` field is where the magic happens. It tells npm, "When someone installs this package, create a command called `commit-genius` that runs `index.js`." It's the difference between users typing `node /path/to/your/tool/index.js` and just typing `commit-genius`. User experience matters, even in the terminal.

The `"type": "module"` is us being modern and using ES6 modules because it's 2024 and we're not barbarians. If you prefer CommonJS, that's fine too, but you'll have to live with yourself.

## The Simplest Possible CLI

Let's start with the absolute minimum:

```javascript
#!/usr/bin/env node

console.log("Analyzing your terrible code...");

const messages = [
  "refactor: improve code quality and reduce technical debt",
  "fix: resolve edge case in production environment",
  "perf: optimize algorithm for better time complexity",
  "feat: implement new feature with backwards compatibility",
  "chore: update dependencies and improve build process"
];

const randomMessage = messages[Math.floor(Math.random() * messages.length)];
console.log(`\nSuggested commit message:\n${randomMessage}`);
```

Save this as `index.js`, make it executable:

```bash
chmod +x index.js
npm link  # This creates a global symlink to your tool
commit-genius  # Look at you, running your own CLI tool!
```

Congratulations! You've built a CLI tool. It's useless, but it works. That's more than most projects can claim.

## Adding Actual Arguments

Now let's make it slightly less useless by accepting some input:

```javascript
#!/usr/bin/env node

const args = process.argv.slice(2);

if (args.length === 0) {
  console.log("Usage: commit-genius <type> [scope]");
  console.log("Types: feat, fix, docs, style, refactor, test, chore");
  process.exit(1);
}

const [type, scope] = args;

const templates = {
  feat: [
    "implement new functionality for enhanced user experience",
    "add feature to improve system capabilities",
    "introduce new component for better modularity"
  ],
  fix: [
    "resolve issue causing unexpected behavior",
    "correct logic error in critical path",
    "fix edge case in error handling"
  ],
  refactor: [
    "improve code structure without changing functionality",
    "optimize implementation for better maintainability",
    "restructure components for improved clarity"
  ]
};

const selectedTemplates = templates[type] || templates.feat;
const randomTemplate = selectedTemplates[Math.floor(Math.random() * selectedTemplates.length)];

const message = scope
  ? `${type}(${scope}): ${randomTemplate}`
  : `${type}: ${randomTemplate}`;

console.log(`\nSuggested commit message:\n${message}`);
```

This is better, but we're already seeing the pain of manual argument parsing. `process.argv` is an array where the first element is the Node.js executable path, the second is your script path, and everything after that is user input. Slicing, validating, parsing—it gets old fast.

## Enter Commander: Making It Professional

Let's upgrade to Commander and make this feel like a real tool:

```javascript
#!/usr/bin/env node

import { Command } from 'commander';
import chalk from 'chalk';

const program = new Command();

const templates = {
  feat: {
    description: "A new feature",
    templates: [
      "implement {description} for enhanced user experience",
      "add {description} to improve system capabilities",
      "introduce {description} for better modularity"
    ]
  },
  fix: {
    description: "A bug fix",
    templates: [
      "resolve {description} causing unexpected behavior",
      "correct {description} in critical path",
      "fix {description} in error handling"
    ]
  },
  refactor: {
    description: "Code change that neither fixes a bug nor adds a feature",
    templates: [
      "improve {description} without changing functionality",
      "optimize {description} for better maintainability",
      "restructure {description} for improved clarity"
    ]
  },
  docs: {
    description: "Documentation only changes",
    templates: [
      "update documentation for {description}",
      "improve docs regarding {description}",
      "clarify documentation about {description}"
    ]
  }
};

program
  .name('commit-genius')
  .description('Generate intelligent commit messages')
  .version('1.0.0');

program
  .command('generate <type> <description>')
  .alias('gen')
  .description('Generate a commit message')
  .option('-s, --scope <scope>', 'add scope to commit message')
  .option('-b, --breaking', 'mark as breaking change')
  .option('-i, --issue <issue>', 'reference issue number')
  .action((type, description, options) => {
    if (!templates[type]) {
      console.error(chalk.red(`Unknown commit type: ${type}`));
      console.log(chalk.yellow('Available types:'), Object.keys(templates).join(', '));
      process.exit(1);
    }

    const templateList = templates[type].templates;
    const randomTemplate = templateList[Math.floor(Math.random() * templateList.length)];
    const body = randomTemplate.replace('{description}', description);

    let message = options.scope
      ? `${type}(${options.scope}): ${body}`
      : `${type}: ${body}`;

    if (options.breaking) {
      message = `${message}\n\nBREAKING CHANGE: This commit contains breaking changes`;
    }

    if (options.issue) {
      message = `${message}\n\nCloses #${options.issue}`;
    }

    console.log(chalk.green('\n✨ Generated commit message:\n'));
    console.log(chalk.cyan(message));
  });

program
  .command('list')
  .description('List all commit types')
  .action(() => {
    console.log(chalk.bold('\nAvailable commit types:\n'));
    Object.entries(templates).forEach(([type, info]) => {
      console.log(chalk.green(`  ${type.padEnd(10)}`), info.description);
    });
  });

program.parse();
```

Now we're cooking! This version has:
- Proper command structure
- Helpful error messages
- Options and flags
- Colored output that makes it feel professional
- Multiple commands (`generate` and `list`)

Try it out:
```bash
commit-genius generate feat "user authentication" --scope auth --issue 42
commit-genius list
commit-genius gen fix "null pointer exception" -s core --breaking
```

## Adding Intelligence with Git Integration

Let's make it actually useful by analyzing real git changes:

```javascript
#!/usr/bin/env node

import { Command } from 'commander';
import chalk from 'chalk';
import { execSync } from 'child_process';
import ora from 'ora';

function getGitDiff() {
  try {
    return execSync('git diff --cached --stat', { encoding: 'utf-8' });
  } catch (error) {
    return null;
  }
}

function getChangedFiles() {
  try {
    const output = execSync('git diff --cached --name-only', { encoding: 'utf-8' });
    return output.trim().split('\n').filter(Boolean);
  } catch (error) {
    return [];
  }
}

function analyzeChanges(files, diff) {
  const analysis = {
    type: 'chore',
    scope: null,
    description: 'update files',
    fileCount: files.length,
    primaryExtension: null
  };

  if (files.length === 0) return analysis;

  // Determine primary file extension
  const extensions = files.map(f => f.split('.').pop());
  const extensionCounts = extensions.reduce((acc, ext) => {
    acc[ext] = (acc[ext] || 0) + 1;
    return acc;
  }, {});

  analysis.primaryExtension = Object.entries(extensionCounts)
    .sort((a, b) => b[1] - a[1])[0][0];

  // Smart type detection
  if (files.some(f => f.includes('test') || f.includes('spec'))) {
    analysis.type = 'test';
    analysis.description = 'update test cases';
  } else if (files.some(f => f.includes('README') || f.includes('.md'))) {
    analysis.type = 'docs';
    analysis.description = 'update documentation';
  } else if (files.some(f => f.includes('package.json'))) {
    analysis.type = 'chore';
    analysis.description = 'update dependencies';
  } else if (diff && diff.includes('+ ') && diff.includes('- ')) {
    const additions = (diff.match(/\+/g) || []).length;
    const deletions = (diff.match(/-/g) || []).length;

    if (deletions > additions) {
      analysis.type = 'refactor';
      analysis.description = 'clean up and optimize code';
    } else {
      analysis.type = 'feat';
      analysis.description = 'add new functionality';
    }
  }

  // Smart scope detection
  if (files.length === 1) {
    const parts = files[0].split('/');
    if (parts.length > 1) {
      analysis.scope = parts[0];
    }
  } else {
    // Find common directory
    const directories = files.map(f => f.split('/')[0]).filter(Boolean);
    const uniqueDirs = [...new Set(directories)];
    if (uniqueDirs.length === 1) {
      analysis.scope = uniqueDirs[0];
    }
  }

  return analysis;
}

const program = new Command();

program
  .name('commit-genius')
  .description('Generate intelligent commit messages based on staged changes')
  .version('2.0.0');

program
  .command('analyze')
  .description('Analyze staged changes and suggest a commit message')
  .option('-v, --verbose', 'show detailed analysis')
  .action((options) => {
    const spinner = ora('Analyzing staged changes...').start();

    const diff = getGitDiff();
    const files = getChangedFiles();

    if (files.length === 0) {
      spinner.fail(chalk.red('No staged changes found'));
      console.log(chalk.yellow('\nTip: Use "git add" to stage your changes first'));
      process.exit(1);
    }

    spinner.succeed('Analysis complete');

    const analysis = analyzeChanges(files, diff);

    if (options.verbose) {
      console.log(chalk.bold('\n📊 Analysis Results:'));
      console.log(chalk.gray('  Files changed:'), analysis.fileCount);
      console.log(chalk.gray('  Primary extension:'), analysis.primaryExtension);
      console.log(chalk.gray('  Detected type:'), analysis.type);
      console.log(chalk.gray('  Detected scope:'), analysis.scope || 'none');
    }

    const message = analysis.scope
      ? `${analysis.type}(${analysis.scope}): ${analysis.description}`
      : `${analysis.type}: ${analysis.description}`;

    console.log(chalk.green('\n✨ Suggested commit message:\n'));
    console.log(chalk.cyan(message));

    console.log(chalk.gray('\n📝 Files to be committed:'));
    files.slice(0, 5).forEach(file => {
      console.log(chalk.gray(`  - ${file}`));
    });
    if (files.length > 5) {
      console.log(chalk.gray(`  ... and ${files.length - 5} more`));
    }

    console.log(chalk.yellow('\nTo use this message:'));
    console.log(chalk.gray(`  git commit -m "${message}"`));
  });

program
  .command('commit')
  .description('Analyze and commit with generated message')
  .option('-e, --edit', 'edit message before committing')
  .option('-y, --yes', 'skip confirmation')
  .action(async (options) => {
    const spinner = ora('Analyzing staged changes...').start();

    const files = getChangedFiles();
    if (files.length === 0) {
      spinner.fail(chalk.red('No staged changes found'));
      process.exit(1);
    }

    const diff = getGitDiff();
    const analysis = analyzeChanges(files, diff);

    const message = analysis.scope
      ? `${analysis.type}(${analysis.scope}): ${analysis.description}`
      : `${analysis.type}: ${analysis.description}`;

    spinner.succeed('Analysis complete');

    console.log(chalk.green('\n✨ Generated commit message:\n'));
    console.log(chalk.cyan(message));

    if (!options.yes) {
      // Here we'd normally use inquirer for confirmation
      console.log(chalk.yellow('\nUse --yes flag to commit automatically'));
      process.exit(0);
    }

    const commitSpinner = ora('Creating commit...').start();

    try {
      if (options.edit) {
        execSync(`git commit -e -m "${message}"`, { stdio: 'inherit' });
      } else {
        execSync(`git commit -m "${message}"`);
      }
      commitSpinner.succeed('Commit created successfully!');
    } catch (error) {
      commitSpinner.fail('Commit failed');
      console.error(chalk.red(error.message));
      process.exit(1);
    }
  });

program.parse();
```

Now we have a tool that actually looks at what you've changed and suggests appropriate commit messages. It's still making educated guesses, but they're educated—like a C+ student who studied the night before.

## Making It Interactive

Let's add some interactivity using Inquirer:

```javascript
#!/usr/bin/env node

import { Command } from 'commander';
import chalk from 'chalk';
import inquirer from 'inquirer';
import { execSync } from 'child_process';
import ora from 'ora';

async function interactiveCommit() {
  const files = getChangedFiles();

  if (files.length === 0) {
    console.log(chalk.red('No staged changes found'));
    return;
  }

  console.log(chalk.bold('\n🎯 Interactive Commit Message Generator\n'));

  const answers = await inquirer.prompt([
    {
      type: 'list',
      name: 'type',
      message: 'Select the type of change:',
      choices: [
        { name: '✨ feat     - A new feature', value: 'feat' },
        { name: '🐛 fix      - A bug fix', value: 'fix' },
        { name: '📝 docs     - Documentation only', value: 'docs' },
        { name: '💅 style    - Code style changes', value: 'style' },
        { name: '♻️  refactor - Code change that neither fixes nor adds', value: 'refactor' },
        { name: '⚡ perf     - Performance improvement', value: 'perf' },
        { name: '✅ test     - Adding missing tests', value: 'test' },
        { name: '🔧 chore    - Maintenance tasks', value: 'chore' }
      ]
    },
    {
      type: 'input',
      name: 'scope',
      message: 'What is the scope of this change? (optional):',
      filter: input => input.trim()
    },
    {
      type: 'input',
      name: 'description',
      message: 'Write a short description:',
      validate: input => {
        if (input.length < 3) {
          return 'Description must be at least 3 characters';
        }
        if (input.length > 50) {
          return 'Description should be less than 50 characters';
        }
        return true;
      }
    },
    {
      type: 'confirm',
      name: 'breaking',
      message: 'Is this a breaking change?',
      default: false
    },
    {
      type: 'input',
      name: 'issue',
      message: 'Issue number (optional):',
      filter: input => input.replace('#', '').trim()
    },
    {
      type: 'editor',
      name: 'body',
      message: 'Provide additional context (optional):',
      when: answers => answers.breaking || answers.issue
    }
  ]);

  // Build commit message
  let message = answers.scope
    ? `${answers.type}(${answers.scope}): ${answers.description}`
    : `${answers.type}: ${answers.description}`;

  if (answers.body) {
    message += `\n\n${answers.body}`;
  }

  if (answers.breaking) {
    message += '\n\nBREAKING CHANGE: This contains breaking changes';
  }

  if (answers.issue) {
    message += `\n\nCloses #${answers.issue}`;
  }

  console.log(chalk.green('\n📝 Your commit message:\n'));
  console.log(chalk.cyan(message));

  const { confirm } = await inquirer.prompt([
    {
      type: 'confirm',
      name: 'confirm',
      message: 'Create commit with this message?',
      default: true
    }
  ]);

  if (confirm) {
    const spinner = ora('Creating commit...').start();
    try {
      // Escape quotes in message for shell
      const escapedMessage = message.replace(/"/g, '\\"');
      execSync(`git commit -m "${escapedMessage}"`);
      spinner.succeed('Commit created successfully!');

      // Show what was committed
      const lastCommit = execSync('git log -1 --oneline', { encoding: 'utf-8' });
      console.log(chalk.gray('\nCreated commit:'), lastCommit.trim());
    } catch (error) {
      spinner.fail('Commit failed');
      console.error(chalk.red(error.message));
    }
  } else {
    console.log(chalk.yellow('Commit cancelled'));
  }
}

function getChangedFiles() {
  try {
    const output = execSync('git diff --cached --name-only', { encoding: 'utf-8' });
    return output.trim().split('\n').filter(Boolean);
  } catch (error) {
    return [];
  }
}

const program = new Command();

program
  .name('commit-genius')
  .description('Generate intelligent commit messages')
  .version('3.0.0');

program
  .command('interactive')
  .alias('i')
  .description('Interactive commit message generator')
  .action(() => {
    interactiveCommit();
  });

// Keep the previous analyze command
program
  .command('analyze')
  .description('Quick analysis of staged changes')
  .action(() => {
    const files = getChangedFiles();

    if (files.length === 0) {
      console.log(chalk.red('No staged changes found'));
      process.exit(1);
    }

    console.log(chalk.bold('\n📊 Staged changes:\n'));

    // Group files by extension
    const byExtension = files.reduce((acc, file) => {
      const ext = file.split('.').pop() || 'no extension';
      acc[ext] = (acc[ext] || []);
      acc[ext].push(file);
      return acc;
    }, {});

    Object.entries(byExtension).forEach(([ext, files]) => {
      console.log(chalk.yellow(`  .${ext} files (${files.length}):`));
      files.slice(0, 3).forEach(file => {
        console.log(chalk.gray(`    - ${file}`));
      });
      if (files.length > 3) {
        console.log(chalk.gray(`    ... and ${files.length - 3} more`));
      }
    });

    // Show statistics
    const stats = execSync('git diff --cached --shortstat', { encoding: 'utf-8' });
    console.log(chalk.bold('\n📈 Statistics:'));
    console.log(chalk.gray(`  ${stats.trim()}`));
  });

program.parse();
```

## Adding Configuration

Every good CLI tool needs configuration. Let's add a config file:

```javascript
import fs from 'fs';
import path from 'path';
import os from 'os';

class Config {
  constructor() {
    this.configPath = path.join(os.homedir(), '.commit-genius.json');
    this.defaults = {
      emoji: true,
      autoAnalyze: true,
      conventionalCommits: true,
      templates: {},
      author: {
        name: '',
        email: ''
      }
    };
    this.config = this.load();
  }

  load() {
    try {
      const data = fs.readFileSync(this.configPath, 'utf-8');
      return { ...this.defaults, ...JSON.parse(data) };
    } catch (error) {
      return this.defaults;
    }
  }

  save() {
    fs.writeFileSync(
      this.configPath,
      JSON.stringify(this.config, null, 2)
    );
  }

  get(key) {
    return key ? this.config[key] : this.config;
  }

  set(key, value) {
    this.config[key] = value;
    this.save();
  }
}

// Add config command to the program
program
  .command('config')
  .description('Configure commit-genius')
  .option('-l, --list', 'list all configuration')
  .option('-s, --set <key=value>', 'set configuration value')
  .option('-g, --get <key>', 'get configuration value')
  .action((options) => {
    const config = new Config();

    if (options.list) {
      console.log(chalk.bold('\n⚙️  Current Configuration:\n'));
      console.log(JSON.stringify(config.get(), null, 2));
    } else if (options.get) {
      const value = config.get(options.get);
      console.log(value);
    } else if (options.set) {
      const [key, value] = options.set.split('=');
      config.set(key, value === 'true' ? true : value === 'false' ? false : value);
      console.log(chalk.green(`✓ Set ${key} to ${value}`));
    } else {
      // Interactive config
      interactiveConfig();
    }
  });

async function interactiveConfig() {
  const config = new Config();

  const answers = await inquirer.prompt([
    {
      type: 'confirm',
      name: 'emoji',
      message: 'Use emojis in commit messages?',
      default: config.get('emoji')
    },
    {
      type: 'confirm',
      name: 'conventionalCommits',
      message: 'Follow conventional commits specification?',
      default: config.get('conventionalCommits')
    },
    {
      type: 'input',
      name: 'author.name',
      message: 'Your name:',
      default: config.get('author').name
    },
    {
      type: 'input',
      name: 'author.email',
      message: 'Your email:',
      default: config.get('author').email,
      validate: input => {
        if (!input || input.includes('@')) return true;
        return 'Please enter a valid email';
      }
    }
  ]);

  Object.entries(answers).forEach(([key, value]) => {
    if (key.includes('.')) {
      const [parent, child] = key.split('.');
      const parentValue = config.get(parent);
      parentValue[child] = value;
      config.set(parent, parentValue);
    } else {
      config.set(key, value);
    }
  });

  console.log(chalk.green('\n✓ Configuration saved!'));
}
```

## Error Handling: When Things Go Wrong

Real CLI tools need to handle errors gracefully:

```javascript
class CommitGenius {
  constructor() {
    this.setupErrorHandlers();
  }

  setupErrorHandlers() {
    process.on('uncaughtException', (error) => {
      console.error(chalk.red('\n💥 Unexpected error:'));
      console.error(chalk.gray(error.stack));
      this.cleanup();
      process.exit(1);
    });

    process.on('unhandledRejection', (reason, promise) => {
      console.error(chalk.red('\n⚠️  Unhandled promise rejection:'));
      console.error(chalk.gray(reason));
      this.cleanup();
      process.exit(1);
    });

    process.on('SIGINT', () => {
      console.log(chalk.yellow('\n\n👋 Interrupted by user'));
      this.cleanup();
      process.exit(0);
    });

    process.on('SIGTERM', () => {
      console.log(chalk.yellow('\n\n🛑 Terminated'));
      this.cleanup();
      process.exit(0);
    });
  }

  cleanup() {
    // Clean up any resources, temp files, etc.
    try {
      // Remove any temporary files
      const tempDir = path.join(os.tmpdir(), 'commit-genius');
      if (fs.existsSync(tempDir)) {
        fs.rmSync(tempDir, { recursive: true });
      }
    } catch (error) {
      // Silently fail cleanup
    }
  }

  checkGitRepository() {
    try {
      execSync('git rev-parse --is-inside-work-tree', {
        encoding: 'utf-8',
        stdio: 'pipe'
      });
      return true;
    } catch (error) {
      console.error(chalk.red('\n❌ Not in a git repository'));
      console.log(chalk.yellow('Initialize a repository with: git init'));
      process.exit(1);
    }
  }

  checkStagedChanges() {
    const files = this.getChangedFiles();
    if (files.length === 0) {
      console.error(chalk.red('\n❌ No staged changes found'));
      console.log(chalk.yellow('Stage changes with: git add <files>'));
      console.log(chalk.gray('Or stage all changes with: git add .'));
      process.exit(1);
    }
    return files;
  }

  run() {
    this.checkGitRepository();
    // Continue with normal operation
  }
}
```

## Making It Fast

Performance matters, even for CLI tools. Here are some tricks:

```javascript
// Lazy loading - only import heavy dependencies when needed
async function loadOra() {
  const { default: ora } = await import('ora');
  return ora;
}

async function showSpinner(text) {
  const ora = await loadOra();
  return ora(text).start();
}

// Parallel operations when possible
async function analyzeRepository() {
  const [files, commits, branches] = await Promise.all([
    getChangedFiles(),
    getRecentCommits(),
    getBranches()
  ]);

  return { files, commits, branches };
}

// Stream processing for large outputs
import { Transform } from 'stream';

class CommitStream extends Transform {
  _transform(chunk, encoding, callback) {
    const lines = chunk.toString().split('\n');

    for (const line of lines) {
      if (line.startsWith('commit ')) {
        this.push(chalk.yellow(line) + '\n');
      } else {
        this.push(line + '\n');
      }
    }

    callback();
  }
}

// Use it for large git logs
const gitLog = spawn('git', ['log', '--oneline', '-n', '1000']);
gitLog.stdout
  .pipe(new CommitStream())
  .pipe(process.stdout);
```

## Distribution: Sharing Your Creation

Once your CLI tool is ready, you need to share it with the world:

```javascript
// Add to package.json
{
  "name": "@yourname/commit-genius",
  "version": "1.0.0",
  "description": "AI-powered commit message generator",
  "main": "index.js",
  "bin": {
    "commit-genius": "./index.js",
    "cg": "./index.js"  // Short alias
  },
  "files": [
    "index.js",
    "lib/**/*",
    "templates/**/*"
  ],
  "engines": {
    "node": ">=14.0.0"
  },
  "preferGlobal": true,
  "repository": {
    "type": "git",
    "url": "https://github.com/yourname/commit-genius.git"
  }
}
```

Publishing is simple:
```bash
npm login
npm publish --access public

# Users can now install globally:
# npm install -g @yourname/commit-genius
```

## The Complete Tool

Let's put it all together into a production-ready CLI:

```javascript
#!/usr/bin/env node

import { Command } from 'commander';
import chalk from 'chalk';
import inquirer from 'inquirer';
import ora from 'ora';
import { execSync, spawn } from 'child_process';
import fs from 'fs';
import path from 'path';
import os from 'os';
import updateNotifier from 'update-notifier';
import pkg from './package.json' assert { type: 'json' };

// Check for updates
updateNotifier({ pkg }).notify();

class CommitGenius {
  constructor() {
    this.program = new Command();
    this.setupCommands();
    this.setupErrorHandlers();
  }

  setupCommands() {
    this.program
      .name('commit-genius')
      .description('AI-powered commit message generator')
      .version(pkg.version)
      .hook('preAction', () => {
        this.checkGitRepository();
      });

    this.program
      .command('analyze')
      .description('Analyze staged changes')
      .option('-v, --verbose', 'verbose output')
      .action(this.analyze.bind(this));

    this.program
      .command('commit')
      .description('Create commit with generated message')
      .option('-i, --interactive', 'interactive mode')
      .option('-y, --yes', 'skip confirmation')
      .action(this.commit.bind(this));

    this.program
      .command('config')
      .description('Configure commit-genius')
      .action(this.config.bind(this));
  }

  async analyze(options) {
    const spinner = ora('Analyzing changes...').start();

    try {
      const analysis = await this.analyzeChanges();
      spinner.succeed('Analysis complete');

      console.log(chalk.bold('\n📊 Analysis Results:'));
      console.log(chalk.gray('  Type:'), analysis.type);
      console.log(chalk.gray('  Scope:'), analysis.scope || 'none');
      console.log(chalk.gray('  Files:'), analysis.fileCount);

      const message = this.generateMessage(analysis);
      console.log(chalk.green('\n✨ Suggested message:'));
      console.log(chalk.cyan(message));
    } catch (error) {
      spinner.fail('Analysis failed');
      throw error;
    }
  }

  async commit(options) {
    if (options.interactive) {
      return this.interactiveCommit();
    }

    const analysis = await this.analyzeChanges();
    const message = this.generateMessage(analysis);

    console.log(chalk.green('\n✨ Commit message:'));
    console.log(chalk.cyan(message));

    if (!options.yes) {
      const { confirm } = await inquirer.prompt([{
        type: 'confirm',
        name: 'confirm',
        message: 'Create commit?',
        default: true
      }]);

      if (!confirm) {
        console.log(chalk.yellow('Cancelled'));
        return;
      }
    }

    this.createCommit(message);
  }

  async interactiveCommit() {
    // Implementation from earlier
  }

  async config() {
    // Configuration implementation
  }

  analyzeChanges() {
    // Analysis logic
  }

  generateMessage(analysis) {
    // Message generation
  }

  createCommit(message) {
    try {
      execSync(`git commit -m "${message.replace(/"/g, '\\"')}"`);
      console.log(chalk.green('✓ Commit created'));
    } catch (error) {
      console.error(chalk.red('Commit failed:'), error.message);
      process.exit(1);
    }
  }

  checkGitRepository() {
    try {
      execSync('git rev-parse --git-dir', { stdio: 'pipe' });
    } catch {
      console.error(chalk.red('Not in a git repository'));
      process.exit(1);
    }
  }

  setupErrorHandlers() {
    process.on('uncaughtException', (error) => {
      console.error(chalk.red('Error:'), error.message);
      if (process.env.DEBUG) {
        console.error(error.stack);
      }
      process.exit(1);
    });
  }

  run() {
    this.program.parse();
  }
}

// Run the CLI
const cli = new CommitGenius();
cli.run();
```

## Lessons Learned

Building a CLI tool teaches you things that web development never will:

1. **Arguments are hard**: What seems simple (passing data to a program) becomes complex quickly. This is why Commander exists.

2. **Cross-platform is harder**: That beautiful spinner that works perfectly on your Mac? It's showing garbage characters on Windows.

3. **Error messages matter**: In a CLI, error messages are your entire UI when things go wrong. Make them helpful.

4. **Performance perception > actual performance**: Users will wait happily for a slow operation if you show them a spinner, but they'll rage-quit if you show nothing for 2 seconds.

5. **Convention matters**: Following conventions (like `-h` for help, `-v` for version) makes your tool immediately familiar.

## The Evolution Path

Your first CLI tool will be simple. Your second will be over-engineered. Your third will find the balance. This is the way.

Start simple, add features as needed, and always remember: the best CLI tool is the one that gets used. It doesn't need to be perfect; it needs to solve a problem. Our commit message generator might seem silly, but it demonstrates every concept you need to build something serious.

In the next chapter, we'll dive deeper into making CLIs interactive, because sometimes a command isn't enough—sometimes you need a conversation. We'll build interfaces that would make GUI developers jealous, all within the confines of a terminal. It's going to be beautiful, in that special way that only terminal applications can be beautiful.