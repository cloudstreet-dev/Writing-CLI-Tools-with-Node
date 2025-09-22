# Chapter 10: Distribution and Updates

You've built the perfect CLI tool. It's tested, optimized, and polished to a shine. Now comes the hard part: getting it into the hands of users and keeping it updated without destroying their workflows. Welcome to the world of software distribution, where semantic versioning is more like semantic suggestion, where "breaking changes" break more than just APIs, and where the phrase "it works on my machine" becomes the most expensive five words in software development.

Distribution isn't just about publishing to npm (though we'll do that too). It's about creating a seamless experience from discovery to installation to daily use. It's about handling updates gracefully, supporting multiple platforms, and dealing with the reality that your users run your tool in environments you've never seen, on machines with architectures you've never heard of, using package managers that may or may not exist by the time they read this sentence.

## Package.json: The Foundation of Everything

Before you can distribute anything, you need a proper `package.json`. This isn't just metadata—it's a contract with the world:

```json
{
  "name": "@yourorg/super-cli",
  "version": "1.0.0",
  "description": "The CLI tool that changes everything (again)",
  "main": "dist/index.js",
  "type": "module",

  "bin": {
    "super-cli": "./dist/cli.js",
    "scli": "./dist/cli.js"
  },

  "engines": {
    "node": ">=16.0.0",
    "npm": ">=7.0.0"
  },

  "os": ["linux", "darwin", "win32"],
  "cpu": ["x64", "arm64"],

  "files": [
    "dist/**/*",
    "README.md",
    "LICENSE",
    "CHANGELOG.md"
  ],

  "scripts": {
    "build": "esbuild src/cli.js --bundle --platform=node --outfile=dist/cli.js",
    "test": "vitest",
    "test:e2e": "npm run build && vitest run e2e",
    "lint": "eslint src/",
    "prepack": "npm run build && npm run test",
    "postinstall": "node scripts/post-install.js",
    "preuninstall": "node scripts/pre-uninstall.js"
  },

  "dependencies": {
    "chalk": "^5.0.0",
    "commander": "^9.0.0",
    "inquirer": "^9.0.0"
  },

  "optionalDependencies": {
    "fsevents": "^2.3.0"
  },

  "peerDependencies": {
    "git": "*"
  },

  "keywords": [
    "cli",
    "developer-tools",
    "productivity",
    "automation"
  ],

  "homepage": "https://super-cli.dev",
  "repository": {
    "type": "git",
    "url": "https://github.com/yourorg/super-cli.git"
  },
  "bugs": {
    "url": "https://github.com/yourorg/super-cli/issues"
  },

  "author": {
    "name": "Your Name",
    "email": "you@yourorg.com",
    "url": "https://yourorg.com"
  },

  "license": "MIT",

  "publishConfig": {
    "access": "public",
    "registry": "https://registry.npmjs.org/"
  }
}
```

## Version Management: The Art of Semantic Versioning

Semantic versioning is simple in theory and chaotic in practice:

```javascript
class VersionManager {
  constructor() {
    this.currentVersion = this.getCurrentVersion();
  }

  getCurrentVersion() {
    return JSON.parse(fs.readFileSync('package.json', 'utf8')).version;
  }

  parseVersion(version) {
    const match = version.match(/^(\d+)\.(\d+)\.(\d+)(?:-(.+))?(?:\+(.+))?$/);
    if (!match) throw new Error(`Invalid version: ${version}`);

    return {
      major: parseInt(match[1]),
      minor: parseInt(match[2]),
      patch: parseInt(match[3]),
      prerelease: match[4] || null,
      buildMetadata: match[5] || null
    };
  }

  bumpVersion(type, prerelease = null) {
    const current = this.parseVersion(this.currentVersion);

    switch (type) {
      case 'major':
        current.major++;
        current.minor = 0;
        current.patch = 0;
        break;
      case 'minor':
        current.minor++;
        current.patch = 0;
        break;
      case 'patch':
        current.patch++;
        break;
      case 'prerelease':
        if (current.prerelease) {
          const match = current.prerelease.match(/^(.+)\.(\d+)$/);
          if (match) {
            current.prerelease = `${match[1]}.${parseInt(match[2]) + 1}`;
          } else {
            current.prerelease = `${current.prerelease}.1`;
          }
        } else {
          current.prerelease = prerelease || 'alpha.1';
        }
        break;
    }

    return this.formatVersion(current);
  }

  formatVersion(version) {
    let v = `${version.major}.${version.minor}.${version.patch}`;
    if (version.prerelease) v += `-${version.prerelease}`;
    if (version.buildMetadata) v += `+${version.buildMetadata}`;
    return v;
  }

  shouldBump(changes) {
    // Analyze changes to determine version bump type
    const hasBreaking = changes.some(c => c.breaking);
    const hasFeatures = changes.some(c => c.type === 'feat');
    const hasFixes = changes.some(c => c.type === 'fix');

    if (hasBreaking) return 'major';
    if (hasFeatures) return 'minor';
    if (hasFixes) return 'patch';
    return null;
  }

  generateChangelog(fromVersion, toVersion) {
    // Generate changelog from git commits
    const commits = this.getCommitsSince(fromVersion);
    const changes = this.parseCommits(commits);

    const changelog = {
      version: toVersion,
      date: new Date().toISOString().split('T')[0],
      changes: {
        breaking: changes.filter(c => c.breaking),
        features: changes.filter(c => c.type === 'feat'),
        fixes: changes.filter(c => c.type === 'fix'),
        improvements: changes.filter(c => c.type === 'perf'),
        docs: changes.filter(c => c.type === 'docs'),
        other: changes.filter(c => !['feat', 'fix', 'perf', 'docs'].includes(c.type))
      }
    };

    return this.formatChangelog(changelog);
  }

  formatChangelog(changelog) {
    let md = `## [${changelog.version}] - ${changelog.date}\n\n`;

    if (changelog.changes.breaking.length > 0) {
      md += '### ⚠️ Breaking Changes\n\n';
      changelog.changes.breaking.forEach(c => {
        md += `- ${c.description}\n`;
      });
      md += '\n';
    }

    if (changelog.changes.features.length > 0) {
      md += '### ✨ Features\n\n';
      changelog.changes.features.forEach(c => {
        md += `- ${c.description}\n`;
      });
      md += '\n';
    }

    if (changelog.changes.fixes.length > 0) {
      md += '### 🐛 Bug Fixes\n\n';
      changelog.changes.fixes.forEach(c => {
        md += `- ${c.description}\n`;
      });
      md += '\n';
    }

    return md;
  }
}

// Automated release workflow
class ReleaseManager {
  constructor() {
    this.versionManager = new VersionManager();
  }

  async createRelease(type) {
    console.log(chalk.bold('🚀 Starting release process...\n'));

    // 1. Check working directory is clean
    await this.ensureCleanWorkingDirectory();

    // 2. Run tests
    await this.runTests();

    // 3. Bump version
    const newVersion = this.versionManager.bumpVersion(type);
    console.log(chalk.green(`📈 Version bumped to ${newVersion}`));

    // 4. Update package.json
    await this.updatePackageVersion(newVersion);

    // 5. Generate changelog
    const changelog = this.versionManager.generateChangelog(
      this.versionManager.currentVersion,
      newVersion
    );
    await this.updateChangelog(changelog);

    // 6. Build
    await this.build();

    // 7. Commit changes
    await this.commitRelease(newVersion);

    // 8. Create git tag
    await this.createTag(newVersion);

    // 9. Push to remote
    await this.pushToRemote(newVersion);

    // 10. Publish to npm
    await this.publishToNpm();

    // 11. Create GitHub release
    await this.createGitHubRelease(newVersion, changelog);

    console.log(chalk.green.bold(`\n🎉 Release ${newVersion} completed successfully!`));
  }

  async ensureCleanWorkingDirectory() {
    const { stdout } = await exec('git status --porcelain');
    if (stdout.trim()) {
      throw new Error('Working directory is not clean. Commit or stash changes first.');
    }
  }

  async runTests() {
    console.log(chalk.blue('🧪 Running tests...'));
    await exec('npm test');
  }

  async build() {
    console.log(chalk.blue('🔨 Building...'));
    await exec('npm run build');
  }

  async commitRelease(version) {
    await exec(`git add package.json CHANGELOG.md`);
    await exec(`git commit -m "chore: release v${version}"`);
  }

  async createTag(version) {
    await exec(`git tag v${version}`);
  }

  async pushToRemote(version) {
    await exec('git push origin main');
    await exec(`git push origin v${version}`);
  }

  async publishToNpm() {
    console.log(chalk.blue('📦 Publishing to npm...'));
    await exec('npm publish');
  }
}
```

## Multi-Platform Distribution

Your users aren't all on macOS. Deal with it:

```javascript
class PlatformBuilder {
  constructor() {
    this.platforms = [
      { os: 'linux', arch: 'x64' },
      { os: 'linux', arch: 'arm64' },
      { os: 'darwin', arch: 'x64' },
      { os: 'darwin', arch: 'arm64' },
      { os: 'win32', arch: 'x64' },
      { os: 'win32', arch: 'arm64' }
    ];
  }

  async buildForAllPlatforms() {
    const builds = [];

    for (const platform of this.platforms) {
      console.log(chalk.blue(`Building for ${platform.os}-${platform.arch}...`));

      const build = await this.buildForPlatform(platform);
      builds.push(build);
    }

    return builds;
  }

  async buildForPlatform({ os, arch }) {
    const outfile = `dist/super-cli-${os}-${arch}${os === 'win32' ? '.exe' : ''}`;

    await esbuild.build({
      entryPoints: ['src/cli.js'],
      bundle: true,
      platform: 'node',
      target: 'node16',
      outfile,
      minify: true,

      // Platform-specific settings
      define: {
        'process.platform': JSON.stringify(os),
        'process.arch': JSON.stringify(arch)
      },

      // Bundle everything except native modules
      external: ['fsevents'],

      // Optimize for target platform
      treeShaking: true,

      banner: {
        js: os === 'win32'
          ? '@echo off\nnode "%~dp0\\super-cli-win32-x64.exe" %*'
          : '#!/usr/bin/env node'
      }
    });

    return {
      platform: `${os}-${arch}`,
      filename: outfile,
      size: (await fs.stat(outfile)).size
    };
  }

  async createInstaller() {
    // Create platform-specific installers
    const installers = {};

    // macOS installer
    installers.macos = await this.createMacOSInstaller();

    // Windows installer
    installers.windows = await this.createWindowsInstaller();

    // Linux packages
    installers.debian = await this.createDebianPackage();
    installers.rpm = await this.createRPMPackage();

    return installers;
  }

  async createMacOSInstaller() {
    // Create .pkg installer for macOS
    const pkgPath = 'dist/super-cli.pkg';

    await exec(`pkgbuild \
      --root dist/macos \
      --identifier com.yourorg.super-cli \
      --version ${this.getVersion()} \
      --install-location /usr/local/bin \
      ${pkgPath}`);

    return pkgPath;
  }

  async createWindowsInstaller() {
    // Create NSIS installer for Windows
    const nsisScript = `
!define APPNAME "Super CLI"
!define COMPANYNAME "Your Org"
!define DESCRIPTION "The CLI tool that changes everything"
!define VERSIONMAJOR 1
!define VERSIONMINOR 0
!define VERSIONBUILD 0

Name "\${APPNAME}"
OutFile "dist/super-cli-installer.exe"
InstallDir "$PROGRAMFILES\\Super CLI"

Section "Super CLI"
  SetOutPath $INSTDIR
  File "dist/super-cli-win32-x64.exe"

  ; Add to PATH
  EnVar::SetHKLM
  EnVar::AddValue "PATH" "$INSTDIR"

  ; Create uninstaller
  WriteUninstaller $INSTDIR\\uninstall.exe
SectionEnd
`;

    await fs.writeFile('installer.nsi', nsisScript);
    await exec('makensis installer.nsi');

    return 'dist/super-cli-installer.exe';
  }

  async createDebianPackage() {
    // Create .deb package for Debian/Ubuntu
    const debDir = 'dist/deb';
    await fs.mkdir(`${debDir}/DEBIAN`, { recursive: true });
    await fs.mkdir(`${debDir}/usr/local/bin`, { recursive: true });

    // Copy binary
    await fs.copyFile('dist/super-cli-linux-x64', `${debDir}/usr/local/bin/super-cli`);
    await fs.chmod(`${debDir}/usr/local/bin/super-cli`, '755');

    // Create control file
    const control = `Package: super-cli
Version: ${this.getVersion()}
Section: utils
Priority: optional
Architecture: amd64
Maintainer: Your Name <you@yourorg.com>
Description: The CLI tool that changes everything
 A powerful command-line interface for modern development.
`;

    await fs.writeFile(`${debDir}/DEBIAN/control`, control);

    // Build package
    await exec(`dpkg-deb --build ${debDir} dist/super-cli.deb`);

    return 'dist/super-cli.deb';
  }
}
```

## Update Mechanisms: Keeping Users Current

Users hate updating software manually. Make it automatic:

```javascript
class UpdateManager {
  constructor(currentVersion) {
    this.currentVersion = currentVersion;
    this.updateCheckUrl = 'https://api.github.com/repos/yourorg/super-cli/releases/latest';
    this.lastCheckFile = path.join(os.homedir(), '.super-cli-update-check');
    this.checkInterval = 24 * 60 * 60 * 1000; // 24 hours
  }

  async checkForUpdates(force = false) {
    // Check if we should check for updates
    if (!force && !await this.shouldCheckForUpdates()) {
      return null;
    }

    try {
      console.log(chalk.gray('Checking for updates...'));

      const response = await fetch(this.updateCheckUrl, {
        headers: {
          'User-Agent': `super-cli/${this.currentVersion}`
        }
      });

      if (!response.ok) {
        throw new Error(`Update check failed: ${response.statusText}`);
      }

      const release = await response.json();
      const latestVersion = release.tag_name.replace(/^v/, '');

      // Update last check time
      await fs.writeFile(this.lastCheckFile, Date.now().toString());

      if (this.isNewerVersion(latestVersion, this.currentVersion)) {
        return {
          currentVersion: this.currentVersion,
          latestVersion,
          releaseNotes: release.body,
          downloadUrl: this.getDownloadUrl(release),
          publishedAt: release.published_at
        };
      }

      return null;
    } catch (error) {
      console.error(chalk.yellow(`Update check failed: ${error.message}`));
      return null;
    }
  }

  async shouldCheckForUpdates() {
    try {
      const lastCheck = await fs.readFile(this.lastCheckFile, 'utf8');
      const lastCheckTime = parseInt(lastCheck);
      return Date.now() - lastCheckTime > this.checkInterval;
    } catch {
      return true; // File doesn't exist, should check
    }
  }

  isNewerVersion(latest, current) {
    const parseVersion = (v) => v.split('.').map(n => parseInt(n));
    const [latestMajor, latestMinor, latestPatch] = parseVersion(latest);
    const [currentMajor, currentMinor, currentPatch] = parseVersion(current);

    if (latestMajor > currentMajor) return true;
    if (latestMajor < currentMajor) return false;
    if (latestMinor > currentMinor) return true;
    if (latestMinor < currentMinor) return false;
    return latestPatch > currentPatch;
  }

  getDownloadUrl(release) {
    const platform = process.platform;
    const arch = process.arch;

    // Find the right asset for current platform
    const asset = release.assets.find(a =>
      a.name.includes(platform) && a.name.includes(arch)
    );

    return asset ? asset.browser_download_url : null;
  }

  async promptForUpdate(updateInfo) {
    console.log(chalk.yellow('\n📦 Update Available!'));
    console.log(chalk.gray(`Current version: ${updateInfo.currentVersion}`));
    console.log(chalk.green(`Latest version:  ${updateInfo.latestVersion}`));
    console.log(chalk.gray(`Published: ${new Date(updateInfo.publishedAt).toLocaleDateString()}`));

    if (updateInfo.releaseNotes) {
      console.log(chalk.blue('\nRelease Notes:'));
      console.log(updateInfo.releaseNotes.slice(0, 300) + '...');
    }

    const { shouldUpdate } = await inquirer.prompt([
      {
        type: 'confirm',
        name: 'shouldUpdate',
        message: 'Would you like to update now?',
        default: true
      }
    ]);

    if (shouldUpdate) {
      await this.performUpdate(updateInfo);
    } else {
      console.log(chalk.gray('Update skipped. Run with --update to update later.'));
    }
  }

  async performUpdate(updateInfo) {
    const spinner = ora('Downloading update...').start();

    try {
      // Download new version
      const tempFile = await this.downloadUpdate(updateInfo.downloadUrl);
      spinner.text = 'Installing update...';

      // Install new version
      await this.installUpdate(tempFile);

      spinner.succeed('Update completed successfully!');
      console.log(chalk.green(`\nUpdated to version ${updateInfo.latestVersion}`));
      console.log(chalk.gray('Please restart your terminal or run the command again.'));
    } catch (error) {
      spinner.fail('Update failed');
      console.error(chalk.red(`Update error: ${error.message}`));
      console.log(chalk.gray('Please update manually or try again later.'));
    }
  }

  async downloadUpdate(url) {
    const response = await fetch(url);
    if (!response.ok) {
      throw new Error(`Download failed: ${response.statusText}`);
    }

    const tempFile = path.join(os.tmpdir(), `super-cli-update-${Date.now()}`);
    const fileStream = fs.createWriteStream(tempFile);

    await new Promise((resolve, reject) => {
      response.body.pipe(fileStream);
      response.body.on('error', reject);
      fileStream.on('finish', resolve);
    });

    return tempFile;
  }

  async installUpdate(tempFile) {
    // Get current executable path
    const currentExe = process.argv[0];
    const backupExe = currentExe + '.backup';

    // Backup current version
    await fs.copyFile(currentExe, backupExe);

    try {
      // Replace with new version
      await fs.copyFile(tempFile, currentExe);
      await fs.chmod(currentExe, '755');

      // Clean up
      await fs.unlink(tempFile);
      await fs.unlink(backupExe);
    } catch (error) {
      // Restore backup on failure
      await fs.copyFile(backupExe, currentExe);
      await fs.unlink(backupExe);
      throw error;
    }
  }

  // Auto-update on startup
  async checkAndNotify() {
    const updateInfo = await this.checkForUpdates();

    if (updateInfo) {
      // Non-intrusive notification
      console.log(chalk.yellow(`\n💡 Super CLI ${updateInfo.latestVersion} is available (you have ${updateInfo.currentVersion})`));
      console.log(chalk.gray('Run with --update to upgrade, or visit https://super-cli.dev for more info.\n'));
    }
  }
}

// Integration with CLI
class SuperCLI {
  constructor() {
    this.updateManager = new UpdateManager(this.getVersion());
  }

  async run() {
    const program = new Command();

    program
      .name('super-cli')
      .version(this.getVersion())
      .option('--update', 'check for and install updates')
      .hook('preAction', async () => {
        // Check for updates on startup (non-blocking)
        setImmediate(() => this.updateManager.checkAndNotify());
      });

    program
      .command('update')
      .description('update to the latest version')
      .action(async () => {
        const updateInfo = await this.updateManager.checkForUpdates(true);
        if (updateInfo) {
          await this.updateManager.promptForUpdate(updateInfo);
        } else {
          console.log(chalk.green('You are already using the latest version!'));
        }
      });

    // Handle --update flag
    if (process.argv.includes('--update')) {
      const updateInfo = await this.updateManager.checkForUpdates(true);
      if (updateInfo) {
        await this.updateManager.promptForUpdate(updateInfo);
        return;
      }
    }

    await program.parseAsync();
  }
}
```

## Alternative Distribution Channels

Not everyone uses npm. Cater to different preferences:

```javascript
class DistributionManager {
  constructor() {
    this.channels = {
      npm: new NPMChannel(),
      homebrew: new HomebrewChannel(),
      scoop: new ScoopChannel(),
      chocolatey: new ChocolateyChannel(),
      snap: new SnapChannel(),
      docker: new DockerChannel(),
      github: new GitHubReleasesChannel()
    };
  }

  async distributeToAll(version, artifacts) {
    const results = {};

    for (const [name, channel] of Object.entries(this.channels)) {
      try {
        console.log(chalk.blue(`📦 Publishing to ${name}...`));
        results[name] = await channel.publish(version, artifacts);
        console.log(chalk.green(`✅ Published to ${name}`));
      } catch (error) {
        console.error(chalk.red(`❌ Failed to publish to ${name}: ${error.message}`));
        results[name] = { error: error.message };
      }
    }

    return results;
  }
}

class HomebrewChannel {
  async publish(version, artifacts) {
    // Create Homebrew formula
    const formula = this.generateFormula(version, artifacts);

    // Submit to homebrew-core or create tap
    await this.submitFormula(formula);

    return { formula, url: 'https://formulae.brew.sh/formula/super-cli' };
  }

  generateFormula(version, artifacts) {
    const macosArtifact = artifacts.find(a => a.platform === 'darwin-x64');

    return `
class SuperCli < Formula
  desc "The CLI tool that changes everything"
  homepage "https://super-cli.dev"
  url "${macosArtifact.downloadUrl}"
  sha256 "${macosArtifact.sha256}"
  version "${version}"

  depends_on "node"

  def install
    bin.install "super-cli"
  end

  test do
    system "#{bin}/super-cli", "--version"
  end
end
`;
  }
}

class DockerChannel {
  async publish(version, artifacts) {
    // Create Dockerfile
    const dockerfile = `
FROM node:18-alpine

WORKDIR /app

COPY dist/super-cli-linux-x64 /usr/local/bin/super-cli
RUN chmod +x /usr/local/bin/super-cli

ENTRYPOINT ["super-cli"]
CMD ["--help"]
`;

    await fs.writeFile('Dockerfile', dockerfile);

    // Build and push Docker image
    await exec(`docker build -t super-cli:${version} .`);
    await exec(`docker tag super-cli:${version} super-cli:latest`);
    await exec(`docker push super-cli:${version}`);
    await exec(`docker push super-cli:latest`);

    return {
      image: `super-cli:${version}`,
      registry: 'docker.io'
    };
  }
}

class SnapChannel {
  async publish(version, artifacts) {
    // Create snapcraft.yaml
    const snapcraft = `
name: super-cli
base: core20
version: '${version}'
summary: The CLI tool that changes everything
description: |
  A powerful command-line interface for modern development workflows.

grade: stable
confinement: strict

parts:
  super-cli:
    plugin: dump
    source: dist/
    organize:
      super-cli-linux-x64: bin/super-cli

apps:
  super-cli:
    command: bin/super-cli
    plugs:
      - home
      - network
`;

    await fs.writeFile('snap/snapcraft.yaml', snapcraft);

    // Build and push snap
    await exec('snapcraft');
    await exec(`snapcraft upload super-cli_${version}_amd64.snap`);

    return {
      snap: `super-cli_${version}_amd64.snap`,
      store: 'https://snapcraft.io/super-cli'
    };
  }
}
```

## Telemetry and Analytics: Understanding Usage

Know how your tool is being used (with user consent):

```javascript
class TelemetryManager {
  constructor() {
    this.enabled = false;
    this.configFile = path.join(os.homedir(), '.super-cli-telemetry');
    this.endpoint = 'https://telemetry.super-cli.dev/events';
    this.userId = this.getUserId();
    this.sessionId = this.generateSessionId();

    this.loadConfig();
  }

  async loadConfig() {
    try {
      const config = JSON.parse(await fs.readFile(this.configFile, 'utf8'));
      this.enabled = config.enabled;
    } catch {
      // Config doesn't exist, prompt user
      await this.promptForConsent();
    }
  }

  async promptForConsent() {
    console.log(chalk.bold('\n📊 Telemetry'));
    console.log(chalk.gray('Super CLI collects anonymous usage data to improve the tool.'));
    console.log(chalk.gray('No personal data or file contents are ever collected.'));
    console.log(chalk.gray('Data collected: command usage, errors, performance metrics, OS/Node version.'));

    const { consent } = await inquirer.prompt([
      {
        type: 'confirm',
        name: 'consent',
        message: 'Enable anonymous telemetry?',
        default: true
      }
    ]);

    this.enabled = consent;
    await this.saveConfig();

    if (consent) {
      console.log(chalk.green('✅ Telemetry enabled. Thank you!'));
    } else {
      console.log(chalk.gray('Telemetry disabled.'));
    }

    console.log(chalk.gray('You can change this anytime with: super-cli telemetry --enable/--disable\n'));
  }

  async saveConfig() {
    const config = {
      enabled: this.enabled,
      userId: this.userId,
      consentedAt: new Date().toISOString()
    };

    await fs.writeFile(this.configFile, JSON.stringify(config, null, 2));
  }

  getUserId() {
    // Generate stable anonymous user ID
    const machineId = os.hostname() + os.userInfo().username;
    return crypto.createHash('sha256').update(machineId).digest('hex').slice(0, 16);
  }

  generateSessionId() {
    return crypto.randomBytes(8).toString('hex');
  }

  async trackEvent(event, properties = {}) {
    if (!this.enabled) return;

    const eventData = {
      event,
      userId: this.userId,
      sessionId: this.sessionId,
      timestamp: new Date().toISOString(),
      properties: {
        ...properties,
        version: this.getVersion(),
        platform: process.platform,
        arch: process.arch,
        nodeVersion: process.version,
        ci: !!process.env.CI
      }
    };

    // Send asynchronously, don't block CLI
    setImmediate(() => this.sendEvent(eventData));
  }

  async sendEvent(eventData) {
    try {
      await fetch(this.endpoint, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'User-Agent': `super-cli/${this.getVersion()}`
        },
        body: JSON.stringify(eventData),
        timeout: 5000
      });
    } catch (error) {
      // Silently fail - telemetry should never break the CLI
    }
  }

  // Track common events
  async trackCommandUsage(command, duration, success, error = null) {
    await this.trackEvent('command_used', {
      command,
      duration,
      success,
      error: error ? error.message : null
    });
  }

  async trackError(error, context = {}) {
    await this.trackEvent('error', {
      error: error.message,
      stack: error.stack?.split('\n')[0], // First line only
      code: error.code,
      ...context
    });
  }

  async trackPerformance(operation, duration, metadata = {}) {
    await this.trackEvent('performance', {
      operation,
      duration,
      ...metadata
    });
  }

  async trackInstallation() {
    await this.trackEvent('installation', {
      installMethod: this.detectInstallMethod()
    });
  }

  detectInstallMethod() {
    // Detect how the CLI was installed
    const execPath = process.argv[0];

    if (execPath.includes('npm') || execPath.includes('node_modules')) {
      return 'npm';
    } else if (execPath.includes('homebrew') || execPath.includes('brew')) {
      return 'homebrew';
    } else if (execPath.includes('scoop')) {
      return 'scoop';
    } else if (execPath.includes('chocolatey')) {
      return 'chocolatey';
    } else {
      return 'manual';
    }
  }
}

// Integration with CLI
class SuperCLI {
  constructor() {
    this.telemetry = new TelemetryManager();
  }

  async run() {
    const startTime = Date.now();
    const command = process.argv[2] || 'help';

    try {
      // Track command start
      await this.telemetry.trackEvent('command_start', { command });

      // Run command
      await this.executeCommand(command);

      // Track success
      const duration = Date.now() - startTime;
      await this.telemetry.trackCommandUsage(command, duration, true);
    } catch (error) {
      // Track failure
      const duration = Date.now() - startTime;
      await this.telemetry.trackCommandUsage(command, duration, false, error);
      await this.telemetry.trackError(error, { command });

      throw error;
    }
  }
}
```

## Usage Analytics Dashboard

Build a simple dashboard to understand your users:

```javascript
// Simple analytics server
import express from 'express';
import { MongoClient } from 'mongodb';

class AnalyticsDashboard {
  constructor() {
    this.app = express();
    this.db = null;
  }

  async start() {
    // Connect to MongoDB
    const client = new MongoClient(process.env.MONGODB_URL);
    await client.connect();
    this.db = client.db('super-cli-analytics');

    // Setup routes
    this.setupRoutes();

    // Start server
    this.app.listen(3000, () => {
      console.log('Analytics dashboard running on http://localhost:3000');
    });
  }

  setupRoutes() {
    this.app.use(express.json());
    this.app.use(express.static('public'));

    // Receive telemetry events
    this.app.post('/events', async (req, res) => {
      try {
        await this.db.collection('events').insertOne({
          ...req.body,
          receivedAt: new Date()
        });
        res.status(200).send('OK');
      } catch (error) {
        res.status(500).send('Error');
      }
    });

    // Analytics API
    this.app.get('/api/stats', async (req, res) => {
      const stats = await this.generateStats();
      res.json(stats);
    });

    this.app.get('/api/commands', async (req, res) => {
      const commands = await this.getCommandUsage();
      res.json(commands);
    });

    this.app.get('/api/errors', async (req, res) => {
      const errors = await this.getErrorStats();
      res.json(errors);
    });

    this.app.get('/api/performance', async (req, res) => {
      const performance = await this.getPerformanceStats();
      res.json(performance);
    });
  }

  async generateStats() {
    const now = new Date();
    const dayAgo = new Date(now - 24 * 60 * 60 * 1000);
    const weekAgo = new Date(now - 7 * 24 * 60 * 60 * 1000);

    const [
      totalUsers,
      activeUsersToday,
      activeUsersWeek,
      totalCommands,
      commandsToday
    ] = await Promise.all([
      this.db.collection('events').distinct('userId').then(users => users.length),
      this.db.collection('events').distinct('userId', { timestamp: { $gte: dayAgo.toISOString() } }).then(users => users.length),
      this.db.collection('events').distinct('userId', { timestamp: { $gte: weekAgo.toISOString() } }).then(users => users.length),
      this.db.collection('events').countDocuments({ event: 'command_used' }),
      this.db.collection('events').countDocuments({
        event: 'command_used',
        timestamp: { $gte: dayAgo.toISOString() }
      })
    ]);

    return {
      totalUsers,
      activeUsersToday,
      activeUsersWeek,
      totalCommands,
      commandsToday
    };
  }

  async getCommandUsage() {
    return this.db.collection('events').aggregate([
      { $match: { event: 'command_used' } },
      { $group: {
        _id: '$properties.command',
        count: { $sum: 1 },
        avgDuration: { $avg: '$properties.duration' },
        successRate: { $avg: { $cond: ['$properties.success', 1, 0] } }
      }},
      { $sort: { count: -1 } },
      { $limit: 20 }
    ]).toArray();
  }

  async getErrorStats() {
    return this.db.collection('events').aggregate([
      { $match: { event: 'error' } },
      { $group: {
        _id: '$properties.error',
        count: { $sum: 1 },
        lastSeen: { $max: '$timestamp' }
      }},
      { $sort: { count: -1 } },
      { $limit: 10 }
    ]).toArray();
  }
}
```

## Distribution Best Practices

After years of distributing CLI tools, here are the hard-won lessons:

### Version Strategy
- Use semantic versioning religiously
- Tag releases consistently
- Maintain a changelog (automate it)
- Never break existing workflows without major version bump

### Platform Support
- Test on all platforms you claim to support
- Windows is different—embrace it, don't fight it
- ARM64 is the future—support it now
- Docker is universal—provide an image

### Update Strategy
- Check for updates, but don't be annoying
- Make updates easy and safe
- Always provide rollback mechanism
- Respect user preferences (some prefer manual updates)

### Telemetry Ethics
- Always ask for consent
- Collect only what you need
- Be transparent about what you collect
- Provide easy opt-out
- Never collect personal data

### Distribution Channels
- NPM for developers
- Homebrew for Mac users
- Chocolatey/Scoop for Windows users
- Package managers for Linux users
- Direct downloads for everyone else

## The Distribution Philosophy

Distribution is marketing. A tool that can't be easily installed won't be used. A tool that breaks on updates won't be trusted. A tool that's hard to find won't be discovered.

Your distribution strategy is as important as your code quality. Maybe more important. The best CLI tool in the world is worthless if nobody can install it.

Think of distribution as the final user experience. The journey from "I heard about this tool" to "I'm using it daily" should be smooth, predictable, and respectful of the user's time and preferences.

Good distribution means:
- Multiple installation methods
- Automatic updates that don't break things
- Clear documentation
- Responsive support
- Respect for user privacy

Remember: users don't care about your clever code. They care about their problems being solved quickly and reliably. Your distribution strategy should reflect that priority.

## Conclusion

We've built CLI tools from the ground up—from argument parsing to AI integration, from file manipulation to performance optimization, from testing strategies to distribution channels. We've explored the ecosystem that makes Node.js the platform of choice for modern command-line tools, and we've seen why tools like Claude Code, Codex CLI, and Gemini CLI have chosen this foundation.

The command line isn't going anywhere. If anything, it's experiencing a renaissance. Developers are realizing that well-designed CLI tools can be more efficient than GUIs for many tasks. AI is making CLI tools smarter and more conversational. The rise of DevOps and infrastructure-as-code has made command-line proficiency essential.

Your CLI tool is part of this evolution. Whether it's a simple utility that saves developers five minutes a day or a complex system that orchestrates entire deployment pipelines, it matters. Every keystroke saved, every error prevented, every process automated makes the development world a little bit better.

The command line is where ideas become reality, where automation lives, and where the real work gets done. Build tools worthy of that responsibility.

Now stop reading and start building. The terminal is waiting.