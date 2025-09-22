# Chapter 7: Building AI-Powered CLIs

The year is 2024, and your CLI tool doesn't just execute commands anymore—it understands them. It doesn't just report errors—it explains them and suggests fixes. It doesn't just process files—it comprehends their content and generates new ones. Welcome to the era of AI-powered CLIs, where tools like Claude Code, Codex CLI, and Gemini CLI are redefining what command-line interfaces can do.

This isn't about slapping ChatGPT onto a bash script and calling it AI-powered. This is about thoughtfully integrating large language models into CLI workflows, managing tokens like they're precious resources (because they are), streaming responses in real-time, and handling the unique challenges that come with probabilistic outputs in a deterministic world.

## The AI Integration Landscape

Before we dive into code, let's understand what we're dealing with. AI in CLI tools typically means one of three things:

1. **Local models**: Running models directly on the user's machine
2. **API-based models**: Calling OpenAI, Anthropic, Google, or other providers
3. **Hybrid approaches**: Using local models for simple tasks, cloud APIs for complex ones

Each approach has trade-offs:

```javascript
class AIProvider {
  constructor(config) {
    this.config = config;
    this.providers = new Map();

    this.initializeProviders();
  }

  initializeProviders() {
    // OpenAI
    if (this.config.openai?.apiKey) {
      this.providers.set('openai', {
        name: 'OpenAI',
        models: ['gpt-4', 'gpt-3.5-turbo', 'gpt-4-vision-preview'],
        endpoint: 'https://api.openai.com/v1',
        headers: {
          'Authorization': `Bearer ${this.config.openai.apiKey}`,
          'Content-Type': 'application/json'
        },
        costPer1kTokens: { input: 0.03, output: 0.06 }, // GPT-4 pricing
        maxTokens: 128000,
        supportsStreaming: true,
        supportsVision: true
      });
    }

    // Anthropic
    if (this.config.anthropic?.apiKey) {
      this.providers.set('anthropic', {
        name: 'Anthropic',
        models: ['claude-3-opus', 'claude-3-sonnet', 'claude-3-haiku'],
        endpoint: 'https://api.anthropic.com/v1',
        headers: {
          'x-api-key': this.config.anthropic.apiKey,
          'anthropic-version': '2023-06-01',
          'Content-Type': 'application/json'
        },
        costPer1kTokens: { input: 0.015, output: 0.075 }, // Claude-3 pricing
        maxTokens: 200000,
        supportsStreaming: true,
        supportsVision: true
      });
    }

    // Local Ollama
    if (this.config.ollama?.enabled) {
      this.providers.set('ollama', {
        name: 'Ollama',
        models: ['llama3', 'mistral', 'codellama', 'phi3'],
        endpoint: 'http://localhost:11434',
        headers: {
          'Content-Type': 'application/json'
        },
        costPer1kTokens: { input: 0, output: 0 }, // Free local inference
        maxTokens: 4096,
        supportsStreaming: true,
        supportsVision: false,
        isLocal: true
      });
    }
  }

  selectProvider(requirements = {}) {
    const {
      needsVision = false,
      needsStreaming = false,
      maxCost = Infinity,
      preferLocal = false,
      minTokens = 0
    } = requirements;

    // Filter providers based on requirements
    const suitable = Array.from(this.providers.values()).filter(provider => {
      if (needsVision && !provider.supportsVision) return false;
      if (needsStreaming && !provider.supportsStreaming) return false;
      if (provider.maxTokens < minTokens) return false;
      if (provider.costPer1kTokens.input > maxCost) return false;
      return true;
    });

    // Sort by preference
    suitable.sort((a, b) => {
      if (preferLocal) {
        if (a.isLocal && !b.isLocal) return -1;
        if (!a.isLocal && b.isLocal) return 1;
      }
      // Sort by cost
      return a.costPer1kTokens.input - b.costPer1kTokens.input;
    });

    return suitable[0] || null;
  }
}
```

## Token Management: The Currency of AI

Tokens are money. Literally. Every token costs something, whether it's API fees or local compute time. Smart CLI tools manage tokens like a miser manages gold:

```javascript
import { encode } from 'gpt-tokenizer';

class TokenManager {
  constructor(options = {}) {
    this.budget = options.budget || 1000000; // Total token budget
    this.used = 0;
    this.cache = new Map();
    this.history = [];
    this.limits = {
      maxContextSize: options.maxContextSize || 100000,
      maxResponseSize: options.maxResponseSize || 4000,
      maxCacheSize: options.maxCacheSize || 50000000 // 50MB
    };
  }

  countTokens(text, model = 'gpt-4') {
    // Use appropriate tokenizer based on model
    // This is simplified - real implementation would use model-specific tokenizers
    return encode(text).length;
  }

  estimateCost(tokens, provider) {
    const inputCost = (tokens.input / 1000) * provider.costPer1kTokens.input;
    const outputCost = (tokens.output / 1000) * provider.costPer1kTokens.output;
    return inputCost + outputCost;
  }

  async buildContext(messages, maxTokens) {
    let totalTokens = 0;
    const includedMessages = [];

    // Always include system message if present
    const systemMessage = messages.find(m => m.role === 'system');
    if (systemMessage) {
      const tokens = this.countTokens(systemMessage.content);
      totalTokens += tokens;
      includedMessages.push(systemMessage);
    }

    // Include messages from most recent, staying under token limit
    for (let i = messages.length - 1; i >= 0; i--) {
      const message = messages[i];
      if (message.role === 'system') continue; // Already included

      const tokens = this.countTokens(message.content);

      if (totalTokens + tokens > maxTokens) {
        // Try to summarize older messages
        const summary = await this.summarizeMessages(
          messages.slice(0, i + 1),
          maxTokens - totalTokens
        );

        if (summary) {
          includedMessages.unshift({
            role: 'assistant',
            content: `[Summary of earlier conversation: ${summary}]`
          });
        }
        break;
      }

      totalTokens += tokens;
      includedMessages.unshift(message);
    }

    return { messages: includedMessages, tokens: totalTokens };
  }

  async summarizeMessages(messages, maxTokens) {
    // Use a smaller model to summarize
    const prompt = `Summarize this conversation in ${Math.floor(maxTokens / 4)} tokens:
${messages.map(m => `${m.role}: ${m.content}`).join('\n')}`;

    // This would call a summarization model
    // For now, return a simple truncation
    const combined = messages.map(m => m.content).join(' ');
    return combined.substring(0, maxTokens * 4); // Rough approximation
  }

  cacheResponse(key, response, tokens) {
    const entry = {
      response,
      tokens,
      timestamp: Date.now(),
      hits: 0
    };

    this.cache.set(key, entry);

    // Enforce cache size limit
    this.enforeCacheLimit();

    return entry;
  }

  getCached(key) {
    const entry = this.cache.get(key);

    if (entry) {
      entry.hits++;
      entry.lastAccessed = Date.now();

      // Move to end (LRU)
      this.cache.delete(key);
      this.cache.set(key, entry);

      return entry.response;
    }

    return null;
  }

  enforeCacheLimit() {
    let totalSize = 0;
    const entries = Array.from(this.cache.entries());

    for (const [key, entry] of entries) {
      const size = JSON.stringify(entry).length;
      totalSize += size;

      if (totalSize > this.limits.maxCacheSize) {
        this.cache.delete(key);
      }
    }
  }

  trackUsage(tokens, cost, provider) {
    this.used += tokens.total;

    this.history.push({
      timestamp: Date.now(),
      tokens,
      cost,
      provider: provider.name,
      remaining: this.budget - this.used
    });

    // Warn if approaching budget
    const percentUsed = (this.used / this.budget) * 100;

    if (percentUsed > 90) {
      console.warn(chalk.red(`⚠️  Token budget ${percentUsed.toFixed(1)}% used`));
    } else if (percentUsed > 75) {
      console.warn(chalk.yellow(`⚠️  Token budget ${percentUsed.toFixed(1)}% used`));
    }
  }

  getUsageStats() {
    const stats = {
      total: this.used,
      budget: this.budget,
      remaining: this.budget - this.used,
      percentUsed: (this.used / this.budget) * 100,
      avgTokensPerRequest: 0,
      totalCost: 0,
      cacheHitRate: 0
    };

    if (this.history.length > 0) {
      stats.avgTokensPerRequest = this.used / this.history.length;
      stats.totalCost = this.history.reduce((sum, h) => sum + h.cost, 0);
    }

    let cacheHits = 0;
    let totalHits = 0;

    for (const entry of this.cache.values()) {
      cacheHits += entry.hits;
      totalHits += entry.hits + 1; // +1 for initial miss
    }

    if (totalHits > 0) {
      stats.cacheHitRate = (cacheHits / totalHits) * 100;
    }

    return stats;
  }
}
```

## Streaming Responses: The Real-Time Experience

Nothing kills the AI magic faster than waiting 30 seconds for a response. Streaming makes AI feel alive:

```javascript
class StreamingAI {
  constructor(provider) {
    this.provider = provider;
    this.decoder = new TextDecoder();
  }

  async *streamCompletion(messages, options = {}) {
    const {
      temperature = 0.7,
      maxTokens = 2000,
      stopSequences = [],
      onToken = null
    } = options;

    const requestBody = this.buildRequestBody(messages, {
      temperature,
      maxTokens,
      stopSequences,
      stream: true
    });

    const response = await fetch(`${this.provider.endpoint}/chat/completions`, {
      method: 'POST',
      headers: this.provider.headers,
      body: JSON.stringify(requestBody)
    });

    if (!response.ok) {
      throw new Error(`API error: ${response.status} ${response.statusText}`);
    }

    const reader = response.body.getReader();
    let buffer = '';
    let totalTokens = 0;

    while (true) {
      const { done, value } = await reader.read();

      if (done) break;

      buffer += this.decoder.decode(value, { stream: true });
      const lines = buffer.split('\n');
      buffer = lines.pop() || '';

      for (const line of lines) {
        if (line.trim() === '' || line.trim() === 'data: [DONE]') {
          continue;
        }

        if (line.startsWith('data: ')) {
          try {
            const data = JSON.parse(line.slice(6));
            const content = this.extractContent(data);

            if (content) {
              totalTokens++;

              if (onToken) {
                onToken(content, totalTokens);
              }

              yield content;
            }
          } catch (error) {
            console.error('Error parsing SSE:', error);
          }
        }
      }
    }
  }

  buildRequestBody(messages, options) {
    // Provider-specific request formatting
    if (this.provider.name === 'OpenAI') {
      return {
        model: options.model || 'gpt-4',
        messages,
        temperature: options.temperature,
        max_tokens: options.maxTokens,
        stop: options.stopSequences,
        stream: options.stream
      };
    } else if (this.provider.name === 'Anthropic') {
      return {
        model: options.model || 'claude-3-sonnet',
        messages,
        temperature: options.temperature,
        max_tokens: options.maxTokens,
        stop_sequences: options.stopSequences,
        stream: options.stream
      };
    }
    // Add other providers...
  }

  extractContent(data) {
    // Provider-specific response parsing
    if (this.provider.name === 'OpenAI') {
      return data.choices?.[0]?.delta?.content || '';
    } else if (this.provider.name === 'Anthropic') {
      return data.delta?.text || '';
    }
    // Add other providers...
  }

  async streamWithUI(messages, options = {}) {
    const spinner = ora('Thinking...').start();
    let response = '';
    let tokenCount = 0;
    let lastUpdate = Date.now();

    try {
      for await (const token of this.streamCompletion(messages, {
        ...options,
        onToken: (content, count) => {
          tokenCount = count;

          // Update spinner periodically
          if (Date.now() - lastUpdate > 100) {
            spinner.text = `Generating... (${tokenCount} tokens)`;
            lastUpdate = Date.now();
          }
        }
      })) {
        response += token;

        // Clear spinner and show response
        if (response.length === 1) {
          spinner.stop();
          process.stdout.write(chalk.cyan(token));
        } else {
          process.stdout.write(chalk.cyan(token));
        }
      }

      process.stdout.write('\n');
      return response;
    } catch (error) {
      spinner.fail('Generation failed');
      throw error;
    }
  }
}
```

## Context Management: The AI's Memory

AI without context is like a goldfish—it forgets everything every three seconds. Good CLI tools maintain context intelligently:

```javascript
class ConversationManager {
  constructor(options = {}) {
    this.conversations = new Map();
    this.maxConversationLength = options.maxLength || 50;
    this.maxConversations = options.maxConversations || 100;
    this.persistPath = options.persistPath || null;

    if (this.persistPath) {
      this.loadConversations();
    }
  }

  createConversation(id = null) {
    const conversationId = id || this.generateId();

    const conversation = {
      id: conversationId,
      messages: [],
      metadata: {
        created: Date.now(),
        updated: Date.now(),
        tokenCount: 0,
        title: null,
        tags: []
      },
      context: {
        files: [],
        codebase: null,
        environment: this.captureEnvironment()
      }
    };

    this.conversations.set(conversationId, conversation);
    this.enforceConversationLimit();

    return conversation;
  }

  addMessage(conversationId, role, content, metadata = {}) {
    const conversation = this.conversations.get(conversationId);

    if (!conversation) {
      throw new Error(`Conversation ${conversationId} not found`);
    }

    const message = {
      role,
      content,
      timestamp: Date.now(),
      tokens: this.countTokens(content),
      ...metadata
    };

    conversation.messages.push(message);
    conversation.metadata.updated = Date.now();
    conversation.metadata.tokenCount += message.tokens;

    // Auto-generate title from first user message
    if (!conversation.metadata.title && role === 'user' && conversation.messages.length === 1) {
      conversation.metadata.title = this.generateTitle(content);
    }

    // Trim old messages if necessary
    this.trimConversation(conversation);

    // Persist if configured
    if (this.persistPath) {
      this.saveConversations();
    }

    return message;
  }

  trimConversation(conversation) {
    while (conversation.messages.length > this.maxConversationLength) {
      const removed = conversation.messages.shift();
      conversation.metadata.tokenCount -= removed.tokens;
    }
  }

  generateTitle(content) {
    // Simple title generation from first message
    const maxLength = 50;
    const cleaned = content.replace(/\n/g, ' ').trim();

    if (cleaned.length <= maxLength) {
      return cleaned;
    }

    return cleaned.substring(0, maxLength - 3) + '...';
  }

  addContext(conversationId, type, data) {
    const conversation = this.conversations.get(conversationId);

    if (!conversation) {
      throw new Error(`Conversation ${conversationId} not found`);
    }

    switch (type) {
      case 'file':
        conversation.context.files.push({
          path: data.path,
          content: data.content,
          language: data.language,
          timestamp: Date.now()
        });
        break;

      case 'codebase':
        conversation.context.codebase = {
          structure: data.structure,
          dependencies: data.dependencies,
          configs: data.configs
        };
        break;

      case 'error':
        if (!conversation.context.errors) {
          conversation.context.errors = [];
        }
        conversation.context.errors.push({
          error: data.error,
          stack: data.stack,
          timestamp: Date.now()
        });
        break;
    }
  }

  buildPromptWithContext(conversationId, userMessage) {
    const conversation = this.conversations.get(conversationId);

    if (!conversation) {
      throw new Error(`Conversation ${conversationId} not found`);
    }

    let contextPrompt = '';

    // Add file context
    if (conversation.context.files.length > 0) {
      contextPrompt += '\nRelevant files:\n';
      for (const file of conversation.context.files.slice(-3)) {  // Last 3 files
        contextPrompt += `\nFile: ${file.path}\n\`\`\`${file.language}\n${file.content}\n\`\`\`\n`;
      }
    }

    // Add codebase context
    if (conversation.context.codebase) {
      contextPrompt += '\nCodebase information:\n';
      contextPrompt += `Dependencies: ${JSON.stringify(conversation.context.codebase.dependencies)}\n`;
    }

    // Add error context
    if (conversation.context.errors?.length > 0) {
      const lastError = conversation.context.errors[conversation.context.errors.length - 1];
      contextPrompt += `\nRecent error:\n${lastError.error}\n${lastError.stack}\n`;
    }

    return contextPrompt + '\n' + userMessage;
  }

  captureEnvironment() {
    return {
      platform: process.platform,
      nodeVersion: process.version,
      cwd: process.cwd(),
      shell: process.env.SHELL,
      terminal: process.env.TERM
    };
  }

  generateId() {
    return `conv_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }

  countTokens(text) {
    // Simplified token counting
    return Math.ceil(text.length / 4);
  }

  enforceConversationLimit() {
    if (this.conversations.size > this.maxConversations) {
      // Remove oldest conversations
      const sorted = Array.from(this.conversations.entries())
        .sort((a, b) => a[1].metadata.updated - b[1].metadata.updated);

      const toRemove = sorted.slice(0, sorted.length - this.maxConversations);

      for (const [id] of toRemove) {
        this.conversations.delete(id);
      }
    }
  }

  async saveConversations() {
    if (!this.persistPath) return;

    const data = {
      version: 1,
      conversations: Array.from(this.conversations.entries()).map(([id, conv]) => ({
        id,
        ...conv
      }))
    };

    await fs.writeFile(this.persistPath, JSON.stringify(data, null, 2));
  }

  async loadConversations() {
    if (!this.persistPath) return;

    try {
      const data = JSON.parse(await fs.readFile(this.persistPath, 'utf8'));

      for (const conv of data.conversations) {
        this.conversations.set(conv.id, conv);
      }
    } catch (error) {
      // File doesn't exist or is corrupted
      console.warn('Could not load conversation history:', error.message);
    }
  }
}
```

## Code Generation and Manipulation

AI-powered CLIs excel at code generation and manipulation. Here's how to do it right:

```javascript
import { parse } from '@babel/parser';
import generate from '@babel/generator';
import traverse from '@babel/traverse';
import * as t from '@babel/types';

class AICodeAssistant {
  constructor(aiProvider) {
    this.ai = aiProvider;
    this.templates = this.loadTemplates();
  }

  async generateCode(specification, options = {}) {
    const {
      language = 'javascript',
      style = 'modern',
      framework = null,
      includeTests = false,
      includeComments = true
    } = options;

    const prompt = this.buildCodeGenerationPrompt(specification, {
      language,
      style,
      framework,
      includeTests,
      includeComments
    });

    const response = await this.ai.complete(prompt, {
      temperature: 0.3,  // Lower temperature for code generation
      maxTokens: 4000
    });

    const code = this.extractCode(response);
    const validated = await this.validateCode(code, language);

    if (!validated.isValid) {
      // Try to fix the code
      const fixed = await this.fixCode(code, validated.errors, language);
      return fixed;
    }

    return code;
  }

  buildCodeGenerationPrompt(specification, options) {
    let prompt = `Generate ${options.language} code with the following requirements:\n\n`;
    prompt += `Specification: ${specification}\n\n`;

    if (options.framework) {
      prompt += `Framework: ${options.framework}\n`;
    }

    prompt += `Style guide:\n`;
    prompt += `- Use ${options.style} ${options.language} syntax\n`;
    prompt += `- ${options.includeComments ? 'Include' : 'Exclude'} explanatory comments\n`;
    prompt += `- Follow best practices and common conventions\n`;
    prompt += `- Handle errors appropriately\n`;

    if (options.includeTests) {
      prompt += `\nAlso generate unit tests for the code.\n`;
    }

    prompt += `\nProvide the code in a markdown code block with proper syntax highlighting.`;

    return prompt;
  }

  extractCode(response) {
    // Extract code from markdown blocks
    const codeBlockRegex = /```(?:\w+)?\n([\s\S]*?)```/g;
    const matches = [];
    let match;

    while ((match = codeBlockRegex.exec(response)) !== null) {
      matches.push(match[1]);
    }

    return matches.join('\n\n');
  }

  async validateCode(code, language) {
    const validators = {
      javascript: this.validateJavaScript.bind(this),
      typescript: this.validateTypeScript.bind(this),
      python: this.validatePython.bind(this),
      // Add more validators
    };

    const validator = validators[language];

    if (!validator) {
      return { isValid: true, errors: [] };
    }

    return validator(code);
  }

  validateJavaScript(code) {
    const errors = [];

    try {
      parse(code, {
        sourceType: 'module',
        plugins: ['jsx', 'typescript'],
        errorRecovery: true
      });
    } catch (error) {
      errors.push({
        type: 'syntax',
        message: error.message,
        line: error.loc?.line,
        column: error.loc?.column
      });
    }

    // Additional validation
    if (!code.includes('export') && !code.includes('module.exports')) {
      errors.push({
        type: 'warning',
        message: 'No exports found. Code may not be usable as a module.'
      });
    }

    return {
      isValid: errors.filter(e => e.type !== 'warning').length === 0,
      errors
    };
  }

  async fixCode(code, errors, language) {
    const errorDescriptions = errors.map(e =>
      `- ${e.type}: ${e.message} at line ${e.line || 'unknown'}`
    ).join('\n');

    const fixPrompt = `Fix the following ${language} code that has errors:\n\n` +
      `\`\`\`${language}\n${code}\n\`\`\`\n\n` +
      `Errors found:\n${errorDescriptions}\n\n` +
      `Provide the corrected code:`;

    const response = await this.ai.complete(fixPrompt, {
      temperature: 0.2,
      maxTokens: 4000
    });

    return this.extractCode(response);
  }

  async refactorCode(code, instructions, options = {}) {
    const {
      preserveAPI = true,
      addTypes = false,
      modernize = true
    } = options;

    const ast = parse(code, {
      sourceType: 'module',
      plugins: ['jsx', 'typescript']
    });

    // Build refactoring prompt
    const prompt = `Refactor the following code according to these instructions:\n${instructions}\n\n` +
      `Requirements:\n` +
      `${preserveAPI ? '- Preserve the external API\n' : ''}` +
      `${addTypes ? '- Add TypeScript types\n' : ''}` +
      `${modernize ? '- Use modern JavaScript features\n' : ''}` +
      `\nOriginal code:\n\`\`\`javascript\n${code}\n\`\`\``;

    const response = await this.ai.complete(prompt, {
      temperature: 0.3,
      maxTokens: 4000
    });

    return this.extractCode(response);
  }

  async explainCode(code, options = {}) {
    const {
      level = 'intermediate',  // beginner, intermediate, expert
      focus = null,  // specific aspect to focus on
      format = 'markdown'
    } = options;

    const prompt = `Explain the following code for a ${level} developer:\n\n` +
      `\`\`\`javascript\n${code}\n\`\`\`\n\n` +
      `${focus ? `Focus on: ${focus}\n` : ''}` +
      `Provide a clear explanation in ${format} format.`;

    const response = await this.ai.complete(prompt, {
      temperature: 0.7,
      maxTokens: 2000
    });

    return response;
  }

  async suggestImprovements(code, options = {}) {
    const analysis = await this.analyzeCode(code);

    const prompt = `Analyze this code and suggest improvements:\n\n` +
      `\`\`\`javascript\n${code}\n\`\`\`\n\n` +
      `Current metrics:\n` +
      `- Complexity: ${analysis.complexity}\n` +
      `- Lines: ${analysis.lines}\n` +
      `- Dependencies: ${analysis.dependencies.length}\n\n` +
      `Suggest specific improvements for:\n` +
      `1. Performance\n` +
      `2. Readability\n` +
      `3. Maintainability\n` +
      `4. Error handling\n` +
      `5. Best practices`;

    const response = await this.ai.complete(prompt, {
      temperature: 0.6,
      maxTokens: 2000
    });

    return this.parseImprovements(response);
  }

  analyzeCode(code) {
    const lines = code.split('\n').length;
    const complexity = this.calculateCyclomaticComplexity(code);
    const dependencies = this.extractDependencies(code);

    return {
      lines,
      complexity,
      dependencies
    };
  }

  calculateCyclomaticComplexity(code) {
    // Simplified complexity calculation
    const conditionals = (code.match(/if|else|switch|case|\?|&&|\|\|/g) || []).length;
    const loops = (code.match(/for|while|do/g) || []).length;
    const functions = (code.match(/function|=>/g) || []).length;

    return conditionals + loops + functions + 1;
  }

  extractDependencies(code) {
    const importRegex = /import .* from ['"](.+)['"]/g;
    const requireRegex = /require\(['"](.+)['"]\)/g;
    const dependencies = new Set();

    let match;
    while ((match = importRegex.exec(code)) !== null) {
      dependencies.add(match[1]);
    }
    while ((match = requireRegex.exec(code)) !== null) {
      dependencies.add(match[1]);
    }

    return Array.from(dependencies);
  }

  parseImprovements(response) {
    const improvements = [];
    const sections = response.split(/\d+\.\s+/);

    for (const section of sections) {
      if (section.trim()) {
        const [category, ...suggestions] = section.split('\n');
        improvements.push({
          category: category.trim(),
          suggestions: suggestions.map(s => s.trim()).filter(Boolean)
        });
      }
    }

    return improvements;
  }

  loadTemplates() {
    return {
      expressAPI: `
const express = require('express');
const app = express();

app.use(express.json());

// Routes
app.get('/api/{resource}', async (req, res) => {
  // Implementation
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
`,
      reactComponent: `
import React, { useState, useEffect } from 'react';

const {ComponentName} = ({ props }) => {
  const [state, setState] = useState(initialState);

  useEffect(() => {
    // Side effects
  }, [dependencies]);

  return (
    <div>
      {/* Component JSX */}
    </div>
  );
};

export default {ComponentName};
`,
      // Add more templates
    };
  }
}
```

## Error Understanding and Resolution

One of the most powerful uses of AI in CLI tools is understanding and fixing errors:

```javascript
class AIErrorResolver {
  constructor(aiProvider) {
    this.ai = aiProvider;
    this.errorPatterns = this.loadErrorPatterns();
    this.resolutionHistory = new Map();
  }

  async diagnoseError(error, context = {}) {
    // Check if we've seen this exact error before
    const cacheKey = this.generateErrorHash(error);
    if (this.resolutionHistory.has(cacheKey)) {
      return this.resolutionHistory.get(cacheKey);
    }

    // Try pattern matching first
    const patternMatch = this.matchErrorPattern(error);
    if (patternMatch) {
      return patternMatch;
    }

    // Use AI for complex diagnosis
    const diagnosis = await this.aiDiagnosis(error, context);
    this.resolutionHistory.set(cacheKey, diagnosis);

    return diagnosis;
  }

  async aiDiagnosis(error, context) {
    const prompt = this.buildDiagnosisPrompt(error, context);

    const response = await this.ai.complete(prompt, {
      temperature: 0.3,
      maxTokens: 2000
    });

    return this.parseDiagnosis(response);
  }

  buildDiagnosisPrompt(error, context) {
    let prompt = `Diagnose and provide solutions for this error:\n\n`;

    prompt += `Error Message: ${error.message}\n`;

    if (error.stack) {
      prompt += `\nStack Trace:\n${error.stack}\n`;
    }

    if (context.code) {
      prompt += `\nRelevant Code:\n\`\`\`${context.language || 'javascript'}\n${context.code}\n\`\`\`\n`;
    }

    if (context.environment) {
      prompt += `\nEnvironment:\n`;
      prompt += `- Platform: ${context.environment.platform}\n`;
      prompt += `- Node Version: ${context.environment.nodeVersion}\n`;
    }

    prompt += `\nProvide:\n`;
    prompt += `1. Root cause explanation\n`;
    prompt += `2. Step-by-step solution\n`;
    prompt += `3. Code fix if applicable\n`;
    prompt += `4. Prevention tips\n`;

    return prompt;
  }

  parseDiagnosis(response) {
    const sections = {
      cause: '',
      solution: [],
      codeFix: '',
      prevention: []
    };

    const lines = response.split('\n');
    let currentSection = null;

    for (const line of lines) {
      if (line.includes('cause') || line.includes('explanation')) {
        currentSection = 'cause';
      } else if (line.includes('solution') || line.includes('fix')) {
        currentSection = 'solution';
      } else if (line.includes('code')) {
        currentSection = 'codeFix';
      } else if (line.includes('prevention') || line.includes('avoid')) {
        currentSection = 'prevention';
      } else if (currentSection) {
        if (currentSection === 'cause') {
          sections.cause += line + '\n';
        } else if (currentSection === 'solution' || currentSection === 'prevention') {
          if (line.trim()) {
            sections[currentSection].push(line.trim());
          }
        } else if (currentSection === 'codeFix') {
          sections.codeFix += line + '\n';
        }
      }
    }

    return sections;
  }

  matchErrorPattern(error) {
    for (const pattern of this.errorPatterns) {
      if (pattern.regex.test(error.message)) {
        return {
          cause: pattern.cause,
          solution: pattern.solution,
          codeFix: pattern.codeFix,
          prevention: pattern.prevention
        };
      }
    }

    return null;
  }

  loadErrorPatterns() {
    return [
      {
        regex: /Cannot find module/,
        cause: 'The specified module is not installed or cannot be found.',
        solution: ['Run npm install', 'Check the module name spelling', 'Verify the import path'],
        codeFix: '',
        prevention: ['Use npm ci for reproducible installs', 'Check package.json dependencies']
      },
      {
        regex: /ENOENT: no such file or directory/,
        cause: 'The file or directory does not exist.',
        solution: ['Verify the file path', 'Create the missing file/directory', 'Check for typos'],
        codeFix: '',
        prevention: ['Use path.join for cross-platform paths', 'Check file existence before accessing']
      },
      // Add more patterns
    ];
  }

  generateErrorHash(error) {
    const key = `${error.message}_${error.code || ''}_${error.stack?.split('\n')[1] || ''}`;
    return crypto.createHash('md5').update(key).digest('hex');
  }

  async suggestFix(error, code) {
    const diagnosis = await this.diagnoseError(error, { code });

    const prompt = `Given this error and diagnosis, provide the fixed code:\n\n` +
      `Original Code:\n\`\`\`javascript\n${code}\n\`\`\`\n\n` +
      `Error: ${error.message}\n` +
      `Diagnosis: ${diagnosis.cause}\n\n` +
      `Provide the corrected code:`;

    const response = await this.ai.complete(prompt, {
      temperature: 0.2,
      maxTokens: 2000
    });

    return this.extractCode(response);
  }

  extractCode(response) {
    const match = response.match(/```(?:\w+)?\n([\s\S]*?)```/);
    return match ? match[1] : response;
  }
}
```

## Real-World Example: An AI-Powered Code Assistant CLI

Let's build a complete AI-powered CLI tool that brings everything together:

```javascript
#!/usr/bin/env node

import { Command } from 'commander';
import chalk from 'chalk';
import inquirer from 'inquirer';
import ora from 'ora';
import fs from 'fs/promises';
import path from 'path';

class AICodeCLI {
  constructor() {
    this.program = new Command();
    this.aiProvider = new AIProvider(this.loadConfig());
    this.conversationManager = new ConversationManager({
      persistPath: path.join(os.homedir(), '.ai-cli-history.json')
    });
    this.codeAssistant = new AICodeAssistant(this.aiProvider);
    this.errorResolver = new AIErrorResolver(this.aiProvider);
    this.currentConversation = null;

    this.setupCommands();
  }

  setupCommands() {
    this.program
      .name('ai-cli')
      .description('AI-powered code assistant CLI')
      .version('1.0.0');

    this.program
      .command('ask <question>')
      .description('Ask a coding question')
      .option('-c, --context <file>', 'include file as context')
      .option('-v, --verbose', 'verbose output')
      .action(this.handleAsk.bind(this));

    this.program
      .command('generate <type>')
      .description('Generate code')
      .option('-o, --output <file>', 'output to file')
      .option('-l, --language <lang>', 'programming language', 'javascript')
      .option('-f, --framework <fw>', 'framework to use')
      .action(this.handleGenerate.bind(this));

    this.program
      .command('fix <file>')
      .description('Fix errors in code')
      .option('-i, --interactive', 'interactive mode')
      .action(this.handleFix.bind(this));

    this.program
      .command('explain <file>')
      .description('Explain code')
      .option('-l, --level <level>', 'explanation level', 'intermediate')
      .action(this.handleExplain.bind(this));

    this.program
      .command('chat')
      .description('Start interactive chat')
      .action(this.handleChat.bind(this));
  }

  async handleAsk(question, options) {
    const spinner = ora('Thinking...').start();

    try {
      // Create or get conversation
      if (!this.currentConversation) {
        this.currentConversation = this.conversationManager.createConversation();
      }

      // Add context if provided
      if (options.context) {
        const content = await fs.readFile(options.context, 'utf8');
        this.conversationManager.addContext(
          this.currentConversation.id,
          'file',
          {
            path: options.context,
            content,
            language: path.extname(options.context).slice(1)
          }
        );
      }

      // Build prompt with context
      const contextualQuestion = this.conversationManager.buildPromptWithContext(
        this.currentConversation.id,
        question
      );

      // Add user message
      this.conversationManager.addMessage(
        this.currentConversation.id,
        'user',
        question
      );

      spinner.stop();

      // Stream response
      const ai = new StreamingAI(this.aiProvider.selectProvider());
      const response = await ai.streamWithUI(
        this.currentConversation.messages,
        { maxTokens: 2000 }
      );

      // Add assistant message
      this.conversationManager.addMessage(
        this.currentConversation.id,
        'assistant',
        response
      );

      if (options.verbose) {
        const stats = this.tokenManager.getUsageStats();
        console.log(chalk.gray(`\nTokens used: ${stats.total}/${stats.budget}`));
      }
    } catch (error) {
      spinner.fail('Failed to get response');
      console.error(chalk.red(error.message));
    }
  }

  async handleGenerate(type, options) {
    const templates = {
      'api': 'RESTful API endpoint',
      'component': 'React component',
      'test': 'Unit tests',
      'cli': 'CLI tool',
      'function': 'Utility function'
    };

    if (!templates[type]) {
      console.error(chalk.red(`Unknown type: ${type}`));
      console.log('Available types:', Object.keys(templates).join(', '));
      return;
    }

    const answers = await inquirer.prompt([
      {
        type: 'input',
        name: 'description',
        message: `Describe the ${templates[type]} you want to generate:`
      },
      {
        type: 'confirm',
        name: 'includeTests',
        message: 'Include unit tests?',
        default: true
      },
      {
        type: 'confirm',
        name: 'includeComments',
        message: 'Include explanatory comments?',
        default: true
      }
    ]);

    const spinner = ora('Generating code...').start();

    try {
      const code = await this.codeAssistant.generateCode(
        `${templates[type]}: ${answers.description}`,
        {
          language: options.language,
          framework: options.framework,
          includeTests: answers.includeTests,
          includeComments: answers.includeComments
        }
      );

      spinner.succeed('Code generated');

      if (options.output) {
        await fs.writeFile(options.output, code);
        console.log(chalk.green(`✓ Saved to ${options.output}`));
      } else {
        console.log('\n' + chalk.cyan(code));
      }
    } catch (error) {
      spinner.fail('Generation failed');
      console.error(chalk.red(error.message));
    }
  }

  async handleFix(file, options) {
    try {
      const code = await fs.readFile(file, 'utf8');
      console.log(chalk.gray(`Analyzing ${file}...`));

      // Try to run the code and capture errors
      const errors = await this.captureErrors(code);

      if (errors.length === 0) {
        console.log(chalk.green('✓ No errors found'));
        return;
      }

      console.log(chalk.red(`Found ${errors.length} error(s):`));

      for (const error of errors) {
        console.log(chalk.red(`  - ${error.message}`));

        const diagnosis = await this.errorResolver.diagnoseError(error, {
          code,
          environment: {
            platform: process.platform,
            nodeVersion: process.version
          }
        });

        console.log(chalk.yellow('\nDiagnosis:'));
        console.log(diagnosis.cause);

        console.log(chalk.yellow('\nSolutions:'));
        diagnosis.solution.forEach(s => console.log(`  - ${s}`));

        if (options.interactive) {
          const { applyFix } = await inquirer.prompt([
            {
              type: 'confirm',
              name: 'applyFix',
              message: 'Apply suggested fix?',
              default: true
            }
          ]);

          if (applyFix) {
            const fixedCode = await this.errorResolver.suggestFix(error, code);
            await fs.writeFile(file, fixedCode);
            console.log(chalk.green(`✓ Fixed and saved ${file}`));
          }
        }
      }
    } catch (error) {
      console.error(chalk.red(`Error: ${error.message}`));
    }
  }

  async handleExplain(file, options) {
    try {
      const code = await fs.readFile(file, 'utf8');
      const spinner = ora('Analyzing code...').start();

      const explanation = await this.codeAssistant.explainCode(code, {
        level: options.level
      });

      spinner.succeed('Analysis complete');
      console.log('\n' + explanation);

      // Offer to suggest improvements
      const { wantImprovements } = await inquirer.prompt([
        {
          type: 'confirm',
          name: 'wantImprovements',
          message: 'Would you like suggestions for improvement?',
          default: true
        }
      ]);

      if (wantImprovements) {
        const improvements = await this.codeAssistant.suggestImprovements(code);
        console.log(chalk.yellow('\n📝 Suggested Improvements:\n'));

        improvements.forEach(imp => {
          console.log(chalk.bold(`${imp.category}:`));
          imp.suggestions.forEach(s => console.log(`  - ${s}`));
        });
      }
    } catch (error) {
      console.error(chalk.red(`Error: ${error.message}`));
    }
  }

  async handleChat() {
    console.log(chalk.bold('\n🤖 AI Code Assistant\n'));
    console.log(chalk.gray('Type "exit" to quit, "clear" to clear context\n'));

    const conversation = this.conversationManager.createConversation();

    while (true) {
      const { input } = await inquirer.prompt([
        {
          type: 'input',
          name: 'input',
          message: chalk.cyan('You:'),
          prefix: ''
        }
      ]);

      if (input.toLowerCase() === 'exit') {
        console.log(chalk.gray('Goodbye!'));
        break;
      }

      if (input.toLowerCase() === 'clear') {
        this.conversationManager.createConversation();
        console.log(chalk.gray('Context cleared'));
        continue;
      }

      // Add user message
      this.conversationManager.addMessage(conversation.id, 'user', input);

      // Get AI response
      console.log(chalk.green('AI: '), '');
      const ai = new StreamingAI(this.aiProvider.selectProvider());
      const response = await ai.streamWithUI(conversation.messages);

      // Add assistant message
      this.conversationManager.addMessage(conversation.id, 'assistant', response);

      console.log();  // Empty line for readability
    }
  }

  async captureErrors(code) {
    // This would actually run the code in a sandboxed environment
    // For demo, we'll parse for syntax errors
    const errors = [];

    try {
      // Try to parse as JavaScript
      require('acorn').parse(code, { ecmaVersion: 2020 });
    } catch (error) {
      errors.push(error);
    }

    return errors;
  }

  loadConfig() {
    // Load configuration from file or environment
    return {
      openai: {
        apiKey: process.env.OPENAI_API_KEY
      },
      anthropic: {
        apiKey: process.env.ANTHROPIC_API_KEY
      },
      ollama: {
        enabled: true
      }
    };
  }

  run() {
    this.program.parse();
  }
}

// Run the CLI
const cli = new AICodeCLI();
cli.run();
```

## The Future of AI-Powered CLIs

We're just scratching the surface. The future holds:

- **Multimodal CLIs**: Understanding screenshots, diagrams, and voice
- **Autonomous agents**: CLIs that can plan and execute complex tasks
- **Collaborative AI**: Multiple AI models working together
- **Learn from usage**: CLIs that improve based on how you use them
- **Context awareness**: Understanding your entire codebase and project history

The key to building great AI-powered CLIs isn't just about integrating AI—it's about doing it thoughtfully. Token management, streaming, context handling, and error recovery aren't optional; they're essential. The AI is powerful, but it's the engineering around it that makes it useful.

In the next chapter, we'll explore performance optimization, because even AI-powered tools need to be fast. We'll learn how to profile, optimize, and squeeze every millisecond of performance out of our Node.js CLI tools. After all, no amount of AI intelligence can make up for a tool that takes 30 seconds to start.