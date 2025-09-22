# Chapter 4: Interactive CLIs and User Experience

Remember when CLIs were just about typing commands and getting text back? Those were simpler times. Now, users expect their terminal applications to have the UX polish of a web app, the responsiveness of a native application, and the intelligence of an AI assistant. They want autocomplete, syntax highlighting, real-time validation, and probably a cup of coffee delivered to their desk. We can't help with the coffee, but we can make terminals feel magical.

## The Psychology of Terminal Interaction

Before we dive into code, let's talk about psychology. Yes, psychology in a programming book. Deal with it.

Humans have expectations when they interact with computers. When they click a button, they expect immediate feedback. When they type, they expect to see characters appear. When something is loading, they expect to see... something. Anything. A terminal that goes silent for three seconds might as well be frozen, crashed, or calculating the meaning of life.

This is why every successful CLI tool has learned the same lesson: perception is reality. Your tool might be doing incredibly complex operations, but if the user sees nothing, they think nothing is happening. Worse, they'll hit Ctrl+C and try again, probably breaking something in the process.

```javascript
// Bad: The silent treatment
async function processFiles(files) {
  for (const file of files) {
    await complexOperation(file);  // User sees nothing for 30 seconds
  }
  console.log('Done!');  // User has already rage-quit
}

// Good: Constant communication
async function processFiles(files) {
  const spinner = ora('Preparing to process files...').start();

  for (let i = 0; i < files.length; i++) {
    spinner.text = `Processing ${files[i]} (${i + 1}/${files.length})`;
    await complexOperation(files[i]);
  }

  spinner.succeed(`Processed ${files.length} files successfully`);
}
```

## The Art of Progressive Disclosure

Not every user needs every option. This is where progressive disclosure comes in—showing simple options first, complex ones later, and expert options only when asked.

```javascript
import inquirer from 'inquirer';
import chalk from 'chalk';

async function setupProject() {
  // Start simple
  const basicAnswers = await inquirer.prompt([
    {
      type: 'input',
      name: 'projectName',
      message: 'Project name:',
      default: 'my-awesome-project'
    },
    {
      type: 'list',
      name: 'projectType',
      message: 'Project type:',
      choices: [
        'Node.js Application',
        'React Frontend',
        'Full Stack Application',
        'CLI Tool'
      ]
    }
  ]);

  // Ask if they want advanced options
  const { wantsAdvanced } = await inquirer.prompt([
    {
      type: 'confirm',
      name: 'wantsAdvanced',
      message: 'Configure advanced options?',
      default: false
    }
  ]);

  let advancedAnswers = {};

  if (wantsAdvanced) {
    advancedAnswers = await inquirer.prompt([
      {
        type: 'checkbox',
        name: 'features',
        message: 'Additional features:',
        choices: [
          { name: 'TypeScript', value: 'typescript' },
          { name: 'ESLint', value: 'eslint' },
          { name: 'Prettier', value: 'prettier' },
          { name: 'Testing (Jest)', value: 'jest' },
          { name: 'Git Hooks (Husky)', value: 'husky' },
          { name: 'CI/CD Pipeline', value: 'ci' }
        ]
      },
      {
        type: 'list',
        name: 'packageManager',
        message: 'Package manager:',
        choices: ['npm', 'yarn', 'pnpm'],
        default: 'npm'
      },
      {
        type: 'input',
        name: 'port',
        message: 'Development server port:',
        default: '3000',
        when: () => basicAnswers.projectType !== 'CLI Tool',
        validate: input => {
          const port = parseInt(input);
          if (isNaN(port)) return 'Please enter a valid port number';
          if (port < 1024) return 'Port should be >= 1024 (unprivileged)';
          if (port > 65535) return 'Port should be <= 65535';
          return true;
        }
      }
    ]);
  }

  return { ...basicAnswers, ...advancedAnswers };
}
```

This approach respects both beginners (who just want to get started) and experts (who want control over everything). It's the terminal equivalent of having both "Express Setup" and "Custom Installation" options.

## Building Conversational Interfaces

The best interactive CLIs feel less like command executors and more like conversations. They remember context, provide intelligent defaults, and guide users through complex processes.

```javascript
class ConversationalCLI {
  constructor() {
    this.context = {};
    this.history = [];
  }

  async start() {
    console.log(chalk.bold('\n👋 Welcome to Project Setup Assistant\n'));
    console.log("I'll help you set up your project. Let's start with some questions.\n");

    await this.gatherContext();
    await this.projectSetup();
    await this.confirmAndCreate();
  }

  async gatherContext() {
    // Gather context about the user and environment
    const { purpose } = await inquirer.prompt([
      {
        type: 'list',
        name: 'purpose',
        message: 'What are you building today?',
        choices: [
          { name: '🌐 A web application', value: 'web' },
          { name: '📱 A mobile app', value: 'mobile' },
          { name: '🤖 An API service', value: 'api' },
          { name: '🛠️  A CLI tool', value: 'cli' },
          { name: '📚 A library/package', value: 'library' },
          { name: '🎮 Something fun!', value: 'fun' }
        ]
      }
    ]);

    this.context.purpose = purpose;

    // Adjust future questions based on context
    if (purpose === 'fun') {
      const { funType } = await inquirer.prompt([
        {
          type: 'list',
          name: 'funType',
          message: 'What kind of fun?',
          choices: [
            '🎮 A game',
            '🎨 Creative coding',
            '🤖 A bot',
            '🎭 An art project'
          ]
        }
      ]);
      this.context.funType = funType;
    }
  }

  async projectSetup() {
    // Use context to provide intelligent defaults
    const suggestions = this.getSuggestions();

    const answers = await inquirer.prompt([
      {
        type: 'input',
        name: 'name',
        message: 'What should we call your project?',
        default: suggestions.name,
        filter: input => input.toLowerCase().replace(/\s+/g, '-'),
        validate: input => {
          if (!/^[a-z0-9-]+$/.test(input)) {
            return 'Project name should only contain lowercase letters, numbers, and hyphens';
          }
          return true;
        }
      },
      {
        type: 'confirm',
        name: 'typescript',
        message: 'Would you like to use TypeScript?',
        default: suggestions.typescript,
        when: () => this.context.purpose !== 'fun'  // Skip for fun projects
      },
      {
        type: 'checkbox',
        name: 'tools',
        message: 'Which tools would you like to include?',
        choices: suggestions.tools,
        when: () => this.context.purpose !== 'fun'
      }
    ]);

    this.context = { ...this.context, ...answers };
  }

  getSuggestions() {
    const { purpose } = this.context;

    const suggestions = {
      web: {
        name: 'my-web-app',
        typescript: true,
        tools: [
          { name: 'React Router', value: 'router', checked: true },
          { name: 'State Management (Redux)', value: 'redux' },
          { name: 'CSS Framework (Tailwind)', value: 'tailwind', checked: true },
          { name: 'Authentication', value: 'auth' },
          { name: 'Database (Prisma)', value: 'database' }
        ]
      },
      api: {
        name: 'my-api-service',
        typescript: true,
        tools: [
          { name: 'Express.js', value: 'express', checked: true },
          { name: 'Database ORM', value: 'orm', checked: true },
          { name: 'Authentication (JWT)', value: 'jwt' },
          { name: 'API Documentation', value: 'swagger' },
          { name: 'Rate Limiting', value: 'ratelimit' }
        ]
      },
      cli: {
        name: 'my-cli-tool',
        typescript: false,
        tools: [
          { name: 'Commander.js', value: 'commander', checked: true },
          { name: 'Chalk (colors)', value: 'chalk', checked: true },
          { name: 'Inquirer (prompts)', value: 'inquirer' },
          { name: 'Update Notifier', value: 'notifier' }
        ]
      },
      fun: {
        name: 'my-fun-project',
        typescript: false,
        tools: []
      }
    };

    return suggestions[purpose] || suggestions.fun;
  }

  async confirmAndCreate() {
    // Show summary with visual hierarchy
    console.log(chalk.bold('\n📋 Project Summary:\n'));

    const tree = this.buildVisualTree();
    console.log(tree);

    const { confirmed } = await inquirer.prompt([
      {
        type: 'confirm',
        name: 'confirmed',
        message: 'Look good? Should I create this project?',
        default: true
      }
    ]);

    if (confirmed) {
      await this.createProject();
    } else {
      console.log(chalk.yellow('\nNo problem! Run me again when you\'re ready.'));
    }
  }

  buildVisualTree() {
    const { name, purpose, typescript, tools = [] } = this.context;

    let tree = '';
    tree += chalk.cyan(`${name}/\n`);
    tree += chalk.gray('├── ') + chalk.yellow('Type: ') + purpose + '\n';

    if (typescript !== undefined) {
      tree += chalk.gray('├── ') + chalk.yellow('TypeScript: ') +
              (typescript ? chalk.green('✓') : chalk.red('✗')) + '\n';
    }

    if (tools.length > 0) {
      tree += chalk.gray('└── ') + chalk.yellow('Tools:\n');
      tools.forEach((tool, index) => {
        const isLast = index === tools.length - 1;
        const prefix = isLast ? '    └── ' : '    ├── ';
        tree += chalk.gray(prefix) + tool + '\n';
      });
    }

    return tree;
  }

  async createProject() {
    const steps = [
      'Creating project directory',
      'Initializing package.json',
      'Installing dependencies',
      'Setting up configuration',
      'Creating initial files'
    ];

    for (const step of steps) {
      const spinner = ora(step).start();
      await this.delay(Math.random() * 1000 + 500);  // Simulate work
      spinner.succeed();
    }

    console.log(chalk.green.bold('\n✨ Project created successfully!\n'));
    console.log(chalk.gray('Next steps:'));
    console.log(chalk.cyan(`  cd ${this.context.name}`));
    console.log(chalk.cyan('  npm start'));
  }

  delay(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

## Real-time Validation and Feedback

Nothing frustrates users more than filling out a long form only to be told at the end that their input was invalid. Real-time validation prevents this:

```javascript
import inquirer from 'inquirer';
import chalk from 'chalk';
import fs from 'fs';
import path from 'path';

// Custom validator that checks in real-time
class RealtimeValidator {
  constructor() {
    this.cache = new Map();
  }

  async validateProjectName(input) {
    // Immediate syntax validation
    if (!input) return 'Project name is required';
    if (!/^[a-z0-9-]+$/.test(input)) {
      return 'Use only lowercase letters, numbers, and hyphens';
    }

    // Check if directory exists (with caching)
    if (this.cache.has(input)) {
      return this.cache.get(input);
    }

    const exists = fs.existsSync(path.join(process.cwd(), input));
    const result = exists
      ? chalk.red(`Directory '${input}' already exists`)
      : true;

    this.cache.set(input, result);
    return result;
  }

  async validateEmail(input) {
    if (!input) return true;  // Optional field

    // Basic format check
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(input)) {
      return 'Please enter a valid email address';
    }

    // Could do async validation here (check if email is already registered, etc.)
    return true;
  }

  async validatePort(input) {
    const port = parseInt(input);

    if (isNaN(port)) {
      return 'Port must be a number';
    }

    if (port < 1 || port > 65535) {
      return 'Port must be between 1 and 65535';
    }

    if (port < 1024) {
      return chalk.yellow('⚠ Ports below 1024 require admin privileges');
    }

    // Check if port is in use
    const isPortFree = await this.checkPort(port);
    if (!isPortFree) {
      return chalk.red(`Port ${port} is already in use`);
    }

    return true;
  }

  async checkPort(port) {
    // In real implementation, would actually check if port is free
    // For demo, just simulate with random result
    return Math.random() > 0.3;
  }
}

// Using the validator
async function setupWithValidation() {
  const validator = new RealtimeValidator();

  const answers = await inquirer.prompt([
    {
      type: 'input',
      name: 'projectName',
      message: 'Project name:',
      validate: input => validator.validateProjectName(input)
    },
    {
      type: 'input',
      name: 'email',
      message: 'Your email (optional):',
      validate: input => validator.validateEmail(input)
    },
    {
      type: 'input',
      name: 'port',
      message: 'Development port:',
      default: '3000',
      validate: input => validator.validatePort(input)
    }
  ]);

  return answers;
}
```

## Autocomplete and Fuzzy Search

Modern users expect autocomplete everywhere. Your CLI should be no different:

```javascript
import inquirer from 'inquirer';
import autocomplete from 'inquirer-autocomplete-prompt';
import fuzzy from 'fuzzy';

inquirer.registerPrompt('autocomplete', autocomplete);

class AutocompleteCLI {
  constructor() {
    this.commands = [
      { name: 'create-react-app', description: 'Create a new React application' },
      { name: 'create-next-app', description: 'Create a Next.js application' },
      { name: 'create-vue-app', description: 'Create a Vue.js application' },
      { name: 'create-express-api', description: 'Create an Express API' },
      { name: 'create-electron-app', description: 'Create an Electron desktop app' },
      { name: 'create-react-native-app', description: 'Create a React Native mobile app' },
      { name: 'create-gatsby-site', description: 'Create a Gatsby static site' },
      { name: 'create-angular-app', description: 'Create an Angular application' }
    ];

    this.packages = null;  // Lazy-loaded from npm
  }

  async selectTemplate() {
    const answer = await inquirer.prompt([
      {
        type: 'autocomplete',
        name: 'template',
        message: 'Select a project template:',
        source: (answers, input) => this.searchTemplates(input),
        pageSize: 10
      }
    ]);

    return answer.template;
  }

  searchTemplates(input = '') {
    return new Promise((resolve) => {
      const results = fuzzy.filter(input, this.commands, {
        extract: item => `${item.name} ${item.description}`
      });

      const formatted = results.map(result => ({
        name: `${chalk.cyan(result.original.name)} - ${chalk.gray(result.original.description)}`,
        value: result.original.name,
        short: result.original.name
      }));

      resolve(formatted);
    });
  }

  async selectPackage() {
    // Load package list from npm (cached)
    if (!this.packages) {
      const spinner = ora('Loading package list...').start();
      this.packages = await this.loadPopularPackages();
      spinner.stop();
    }

    const answer = await inquirer.prompt([
      {
        type: 'autocomplete',
        name: 'package',
        message: 'Search for npm package:',
        source: (answers, input) => this.searchPackages(input),
        pageSize: 10
      }
    ]);

    return answer.package;
  }

  async loadPopularPackages() {
    // In real implementation, this would fetch from npm
    // For demo, return mock data
    return [
      'express', 'react', 'lodash', 'axios', 'moment',
      'chalk', 'inquirer', 'commander', 'dotenv', 'uuid',
      'bcrypt', 'jsonwebtoken', 'mongoose', 'sequelize'
    ];
  }

  searchPackages(input = '') {
    return new Promise((resolve) => {
      if (input.length < 2) {
        resolve([{
          name: chalk.gray('Type at least 2 characters to search...'),
          value: null
        }]);
        return;
      }

      // Simulate API delay
      setTimeout(() => {
        const results = fuzzy.filter(input, this.packages);
        const formatted = results.map(result => ({
          name: result.string,
          value: result.string
        }));

        if (formatted.length === 0) {
          formatted.push({
            name: chalk.yellow(`No packages found matching "${input}"`),
            value: null
          });
        }

        resolve(formatted);
      }, 200);
    });
  }
}
```

## Multi-Step Wizards

Complex operations need guided workflows. Here's how to build a wizard that maintains state across steps:

```javascript
class SetupWizard {
  constructor() {
    this.steps = [];
    this.currentStep = 0;
    this.state = {};
    this.history = [];
  }

  addStep(step) {
    this.steps.push(step);
    return this;
  }

  async run() {
    console.clear();
    console.log(chalk.bold('🧙 Setup Wizard\n'));

    while (this.currentStep < this.steps.length) {
      const step = this.steps[this.currentStep];

      // Show progress
      this.showProgress();

      // Run step
      const result = await this.runStep(step);

      if (result === 'back' && this.currentStep > 0) {
        this.currentStep--;
        this.state = this.history.pop();
      } else if (result === 'cancel') {
        console.log(chalk.yellow('\n✖ Setup cancelled'));
        return null;
      } else {
        this.history.push({ ...this.state });
        this.state = { ...this.state, ...result };
        this.currentStep++;
      }

      console.clear();
    }

    return this.state;
  }

  showProgress() {
    const progress = this.steps.map((step, index) => {
      if (index < this.currentStep) {
        return chalk.green('●');  // Completed
      } else if (index === this.currentStep) {
        return chalk.cyan('●');  // Current
      } else {
        return chalk.gray('○');  // Upcoming
      }
    }).join(' ');

    console.log(progress);
    console.log(chalk.gray(`Step ${this.currentStep + 1} of ${this.steps.length}`));
    console.log('');
  }

  async runStep(step) {
    console.log(chalk.bold(step.title));
    if (step.description) {
      console.log(chalk.gray(step.description));
    }
    console.log('');

    const questions = typeof step.questions === 'function'
      ? step.questions(this.state)
      : step.questions;

    // Add navigation options
    const questionsWithNav = [
      ...questions,
      {
        type: 'list',
        name: '_navigation',
        message: '',
        choices: () => {
          const choices = [];
          if (this.currentStep > 0) choices.push({ name: '← Back', value: 'back' });
          choices.push({ name: '→ Continue', value: 'continue' });
          choices.push({ name: '✖ Cancel', value: 'cancel' });
          return choices;
        },
        default: 'continue'
      }
    ];

    const answers = await inquirer.prompt(questionsWithNav);

    if (answers._navigation === 'back') return 'back';
    if (answers._navigation === 'cancel') return 'cancel';

    delete answers._navigation;
    return answers;
  }
}

// Using the wizard
async function runProjectWizard() {
  const wizard = new SetupWizard();

  wizard
    .addStep({
      title: 'Project Basics',
      description: 'Let\'s start with the basic information',
      questions: [
        {
          type: 'input',
          name: 'name',
          message: 'Project name:',
          validate: input => input.length > 0
        },
        {
          type: 'input',
          name: 'description',
          message: 'Project description:'
        }
      ]
    })
    .addStep({
      title: 'Technology Stack',
      description: 'Choose your technologies',
      questions: (state) => [
        {
          type: 'list',
          name: 'framework',
          message: 'Frontend framework:',
          choices: ['React', 'Vue', 'Angular', 'None']
        },
        {
          type: 'checkbox',
          name: 'features',
          message: 'Additional features:',
          choices: (answers) => {
            const choices = ['TypeScript', 'ESLint', 'Prettier'];
            if (answers.framework !== 'None') {
              choices.push('Router', 'State Management');
            }
            return choices;
          }
        }
      ]
    })
    .addStep({
      title: 'Configuration',
      description: 'Final configuration options',
      questions: (state) => [
        {
          type: 'confirm',
          name: 'git',
          message: 'Initialize git repository?',
          default: true
        },
        {
          type: 'confirm',
          name: 'install',
          message: 'Install dependencies now?',
          default: true
        }
      ]
    });

  const result = await wizard.run();
  return result;
}
```

## Terminal UI Components

Sometimes you need more than prompts. You need a full UI:

```javascript
import blessed from 'blessed';

class TerminalDashboard {
  constructor() {
    this.screen = blessed.screen({
      smartCSR: true,
      title: 'CLI Dashboard'
    });

    this.setupLayout();
    this.setupEventHandlers();
  }

  setupLayout() {
    // Header
    this.header = blessed.box({
      top: 0,
      left: 0,
      width: '100%',
      height: 3,
      content: '{center}🎯 Project Dashboard{/center}',
      tags: true,
      style: {
        fg: 'white',
        bg: 'blue'
      }
    });

    // Menu
    this.menu = blessed.list({
      top: 3,
      left: 0,
      width: '30%',
      height: '100%-3',
      label: ' Menu ',
      border: {
        type: 'line'
      },
      style: {
        selected: {
          bg: 'cyan',
          fg: 'black'
        }
      },
      keys: true,
      vi: true,
      items: [
        '📊 Overview',
        '📁 Files',
        '⚙️  Settings',
        '📈 Statistics',
        '🔧 Tools',
        '❓ Help'
      ]
    });

    // Main content area
    this.content = blessed.box({
      top: 3,
      left: '30%',
      width: '70%',
      height: '50%',
      label: ' Content ',
      content: '',
      border: {
        type: 'line'
      },
      scrollable: true,
      alwaysScroll: true,
      keys: true,
      vi: true,
      style: {
        fg: 'white'
      }
    });

    // Log area
    this.log = blessed.log({
      top: '50%+3',
      left: '30%',
      width: '70%',
      height: '50%-3',
      label: ' Log ',
      border: {
        type: 'line'
      },
      scrollable: true,
      alwaysScroll: true,
      style: {
        fg: 'green'
      }
    });

    // Add all elements to screen
    this.screen.append(this.header);
    this.screen.append(this.menu);
    this.screen.append(this.content);
    this.screen.append(this.log);

    // Focus on menu
    this.menu.focus();
  }

  setupEventHandlers() {
    // Menu selection
    this.menu.on('select', (item, index) => {
      this.handleMenuSelect(index);
    });

    // Keyboard shortcuts
    this.screen.key(['escape', 'q', 'C-c'], () => {
      return process.exit(0);
    });

    this.screen.key(['tab'], () => {
      if (this.menu.focused) {
        this.content.focus();
      } else {
        this.menu.focus();
      }
      this.screen.render();
    });

    // Handle resize
    this.screen.on('resize', () => {
      this.header.emit('attach');
      this.menu.emit('attach');
      this.content.emit('attach');
      this.log.emit('attach');
    });
  }

  handleMenuSelect(index) {
    const content = [
      this.getOverviewContent(),
      this.getFilesContent(),
      this.getSettingsContent(),
      this.getStatisticsContent(),
      this.getToolsContent(),
      this.getHelpContent()
    ];

    this.content.setContent(content[index]);
    this.log.log(`Selected: ${this.menu.items[index].content}`);
    this.screen.render();
  }

  getOverviewContent() {
    return `
{bold}Project Overview{/bold}

Name: my-awesome-project
Type: Node.js Application
Version: 1.0.0
Dependencies: 42
Dev Dependencies: 18

{yellow-fg}Status:{/yellow-fg} All systems operational
{green-fg}Tests:{/green-fg} 24/24 passing
{cyan-fg}Coverage:{/cyan-fg} 87%
    `.trim();
  }

  getFilesContent() {
    return `
{bold}Project Structure{/bold}

📁 src/
  📄 index.js
  📄 app.js
  📁 components/
    📄 Header.js
    📄 Footer.js
  📁 utils/
    📄 helpers.js
    📄 constants.js
📁 tests/
  📄 app.test.js
📄 package.json
📄 README.md
    `.trim();
  }

  getStatisticsContent() {
    // Create ASCII chart
    const data = [30, 50, 80, 40, 60, 90, 70];
    const maxValue = Math.max(...data);
    const chart = data.map((value, i) => {
      const height = Math.round((value / maxValue) * 10);
      const bar = '█'.repeat(height);
      return `Day ${i + 1}: ${bar} ${value}`;
    }).join('\n');

    return `
{bold}Weekly Activity{/bold}

${chart}

{bold}Totals{/bold}
Commits: 47
Lines Added: 1,284
Lines Removed: 392
Files Changed: 23
    `.trim();
  }

  getSettingsContent() {
    return 'Settings panel - configure your dashboard here';
  }

  getToolsContent() {
    return 'Tools panel - quick access to common tools';
  }

  getHelpContent() {
    return `
{bold}Keyboard Shortcuts{/bold}

Tab       - Switch focus between panels
Enter     - Select menu item
j/k       - Navigate up/down (vi mode)
q/ESC     - Quit application
/         - Search (in focused panel)
    `.trim();
  }

  run() {
    this.screen.render();
  }
}

// Usage
const dashboard = new TerminalDashboard();
dashboard.run();
```

## Responsive Terminal Design

Terminals come in all sizes. Your CLI needs to adapt:

```javascript
import terminalSize from 'terminal-size';
import wrapAnsi from 'wrap-ansi';

class ResponsiveOutput {
  constructor() {
    this.updateSize();

    // Listen for terminal resize
    process.stdout.on('resize', () => {
      this.updateSize();
    });
  }

  updateSize() {
    const { columns, rows } = terminalSize();
    this.width = columns;
    this.height = rows;
  }

  print(text, options = {}) {
    const wrapped = wrapAnsi(text, this.width - 4, options);
    console.log(wrapped);
  }

  printColumns(items, minColumnWidth = 20) {
    const columnCount = Math.floor(this.width / minColumnWidth);
    const columnWidth = Math.floor(this.width / columnCount);

    const rows = [];
    for (let i = 0; i < items.length; i += columnCount) {
      const row = items
        .slice(i, i + columnCount)
        .map(item => item.padEnd(columnWidth))
        .join('');
      rows.push(row);
    }

    rows.forEach(row => console.log(row));
  }

  printTable(data) {
    if (this.width < 60) {
      // Vertical layout for narrow terminals
      data.forEach(row => {
        console.log(chalk.bold(row.name));
        console.log(chalk.gray(`  ${row.description}`));
        console.log(chalk.gray(`  Status: ${row.status}`));
        console.log('');
      });
    } else {
      // Table layout for wide terminals
      console.table(data);
    }
  }

  printProgress(current, total, label = '') {
    const percentage = (current / total) * 100;
    const availableWidth = this.width - label.length - 15;
    const filledWidth = Math.round((percentage / 100) * availableWidth);
    const emptyWidth = availableWidth - filledWidth;

    const bar = chalk.green('█').repeat(filledWidth) +
                chalk.gray('░').repeat(emptyWidth);

    const output = `${label} ${bar} ${percentage.toFixed(1)}%`;
    process.stdout.write(`\r${output}`);
  }
}
```

## Animation and Motion

A little animation goes a long way in making CLIs feel alive:

```javascript
class AnimatedCLI {
  async typeWriter(text, delay = 50) {
    for (const char of text) {
      process.stdout.write(char);
      await this.sleep(delay);
    }
    console.log();
  }

  async loadingDots(message, duration = 3000) {
    const frames = ['.  ', '.. ', '...', '   '];
    let frame = 0;
    const interval = 250;
    const iterations = duration / interval;

    for (let i = 0; i < iterations; i++) {
      process.stdout.write(`\r${message}${frames[frame % frames.length]}`);
      frame++;
      await this.sleep(interval);
    }

    process.stdout.write(`\r${message}...Done!\n`);
  }

  async slideIn(text, direction = 'left') {
    const width = process.stdout.columns;
    const lines = text.split('\n');

    for (const line of lines) {
      if (direction === 'left') {
        for (let i = 0; i <= line.length; i++) {
          process.stdout.write(`\r${line.substring(0, i)}`);
          await this.sleep(10);
        }
      } else {
        for (let i = width; i >= 0; i--) {
          const spaces = ' '.repeat(i);
          process.stdout.write(`\r${spaces}${line}`);
          await this.sleep(10);
        }
      }
      console.log();
    }
  }

  async matrixRain(duration = 5000) {
    const width = process.stdout.columns;
    const height = process.stdout.rows;
    const columns = Array(width).fill(0);

    console.clear();
    console.log('\x1b[?25l');  // Hide cursor

    const interval = setInterval(() => {
      let output = '';

      for (let y = 0; y < height; y++) {
        for (let x = 0; x < width; x++) {
          if (columns[x] > y) {
            const char = String.fromCharCode(33 + Math.random() * 93);
            const brightness = (columns[x] - y) / 20;
            const green = Math.floor(255 * Math.min(1, brightness));
            output += `\x1b[38;2;0;${green};0m${char}\x1b[0m`;
          } else {
            output += ' ';
          }
        }
        if (y < height - 1) output += '\n';
      }

      process.stdout.write('\x1b[H' + output);  // Move cursor to home

      // Update columns
      for (let x = 0; x < width; x++) {
        if (Math.random() > 0.95) {
          columns[x] = 0;
        } else {
          columns[x]++;
        }
      }
    }, 50);

    setTimeout(() => {
      clearInterval(interval);
      console.log('\x1b[?25h');  // Show cursor
      console.clear();
    }, duration);
  }

  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

## Error States and Recovery

Good UX means handling errors gracefully and helping users recover:

```javascript
class ErrorHandler {
  handleError(error, context = {}) {
    console.log();  // Empty line for spacing

    // Determine error type and severity
    const errorInfo = this.classifyError(error);

    // Display error based on severity
    this[`display${errorInfo.severity}Error`](errorInfo);

    // Provide recovery suggestions
    this.suggestRecovery(errorInfo, context);

    // Offer next actions
    this.offerActions(errorInfo, context);
  }

  classifyError(error) {
    if (error.code === 'ENOENT') {
      return {
        severity: 'Warning',
        type: 'file_not_found',
        message: 'File or directory not found',
        details: error.path
      };
    }

    if (error.code === 'EACCES') {
      return {
        severity: 'Error',
        type: 'permission_denied',
        message: 'Permission denied',
        details: error.path
      };
    }

    if (error.message.includes('network')) {
      return {
        severity: 'Warning',
        type: 'network',
        message: 'Network issue detected',
        details: error.message
      };
    }

    return {
      severity: 'Error',
      type: 'unknown',
      message: error.message || 'An unexpected error occurred',
      details: error.stack
    };
  }

  displayWarningError(info) {
    console.log(chalk.yellow('⚠ Warning: ') + info.message);
    if (info.details) {
      console.log(chalk.gray(`  ${info.details}`));
    }
  }

  displayErrorError(info) {
    console.log(chalk.red('✖ Error: ') + info.message);
    if (info.details && process.env.DEBUG) {
      console.log(chalk.gray(info.details));
    }
  }

  suggestRecovery(errorInfo, context) {
    console.log();
    console.log(chalk.bold('💡 Suggestions:'));

    const suggestions = {
      file_not_found: [
        `Check if the file exists: ls ${path.dirname(errorInfo.details)}`,
        `Create the file: touch ${errorInfo.details}`,
        'Verify the path is correct'
      ],
      permission_denied: [
        `Try with sudo: sudo ${context.command}`,
        `Check file permissions: ls -la ${errorInfo.details}`,
        `Change permissions: chmod 755 ${errorInfo.details}`
      ],
      network: [
        'Check your internet connection',
        'Try again in a few moments',
        'Check if you\'re behind a proxy',
        'Verify the API endpoint is correct'
      ],
      unknown: [
        'Try running with --debug flag for more information',
        'Check the documentation: npm docs <package>',
        'Report an issue: https://github.com/your/repo/issues'
      ]
    };

    const relevantSuggestions = suggestions[errorInfo.type] || suggestions.unknown;
    relevantSuggestions.forEach(suggestion => {
      console.log(chalk.gray(`  • ${suggestion}`));
    });
  }

  async offerActions(errorInfo, context) {
    console.log();

    const actions = this.getAvailableActions(errorInfo, context);

    if (actions.length === 0) return;

    const { action } = await inquirer.prompt([
      {
        type: 'list',
        name: 'action',
        message: 'What would you like to do?',
        choices: actions
      }
    ]);

    await this.executeAction(action, errorInfo, context);
  }

  getAvailableActions(errorInfo, context) {
    const actions = [];

    actions.push({ name: 'Try again', value: 'retry' });
    actions.push({ name: 'Show more details', value: 'details' });

    if (errorInfo.type === 'file_not_found') {
      actions.push({ name: 'Create the file', value: 'create' });
    }

    if (errorInfo.type === 'permission_denied') {
      actions.push({ name: 'Try with sudo', value: 'sudo' });
    }

    actions.push({ name: 'Exit', value: 'exit' });

    return actions;
  }

  async executeAction(action, errorInfo, context) {
    switch (action) {
      case 'retry':
        console.log(chalk.cyan('Retrying...'));
        // Retry the operation
        break;
      case 'details':
        console.log(chalk.gray(errorInfo.details));
        break;
      case 'create':
        console.log(chalk.cyan(`Creating ${errorInfo.details}...`));
        // Create the file
        break;
      case 'sudo':
        console.log(chalk.cyan('Retrying with elevated permissions...'));
        // Retry with sudo
        break;
      case 'exit':
        process.exit(1);
    }
  }
}
```

## The UX Principles

After all this code, let's distill the principles of good CLI UX:

1. **Immediate Feedback**: Never leave users wondering if something is happening
2. **Progressive Disclosure**: Show simple options first, complex ones when needed
3. **Graceful Degradation**: Work in all terminals, look great in modern ones
4. **Error Recovery**: Help users fix problems, don't just report them
5. **Contextual Help**: Provide help where and when it's needed
6. **Respect User Time**: Cache responses, remember preferences, provide shortcuts
7. **Visual Hierarchy**: Use color, spacing, and symbols to guide attention

## The Future of Terminal UX

We're entering a golden age of terminal interfaces. AI assistants like Claude Code are showing us that terminals can be conversational. Tools are becoming more intelligent, more responsive, and more delightful to use.

The terminal is no longer just a place to type commands—it's a canvas for creating powerful, beautiful, and intuitive interfaces. The constraints that once limited us (text-only, limited colors) have become the creative boundaries that inspire innovation.

In the next chapter, we'll dive into file system operations, because at the end of the day, most CLI tools are really just fancy ways to manipulate files. But as we've seen, there's nothing simple about making complex operations feel simple. That's the art of CLI development—hiding complexity behind elegant interfaces that just work.