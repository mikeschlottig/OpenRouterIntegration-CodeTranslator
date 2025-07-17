OpenRouter AI Integration

AI-Powered Template Development Framework Chat
https://claude.ai/chat/3d8099de-1f06-4544-ad67-57bc9c4f51d8

Code Context Application Prototype

https://claude.ai/public/artifacts/91e42de5-35a4-4624-b8ad-d5ee44b35738

import React, { useState, useEffect, useCallback, useRef } from 'react';
import { MessageCircle, Send, Settings, Key, Zap, AlertCircle, CheckCircle, Copy, RefreshCw } from 'lucide-react';

// Types and Interfaces
interface OpenRouterConfig {
  apiKey: string;
  model: string;
  baseUrl: string;
  temperature: number;
  maxTokens: number;
  topP: number;
}

interface Message {
  id: string;
  role: 'user' | 'assistant' | 'system';
  content: string;
  timestamp: Date;
  tokens?: number;
  cost?: number;
}

interface OpenRouterResponse {
  id: string;
  object: string;
  created: number;
  model: string;
  choices: Array<{
    index: number;
    message: {
      role: string;
      content: string;
    };
    finish_reason: string;
  }>;
  usage: {
    prompt_tokens: number;
    completion_tokens: number;
    total_tokens: number;
  };
}

interface CodeContext {
  fileName: string;
  fileContent: string;
  language: string;
  framework: string;
  dependencies: string[];
}

// OpenRouter API Integration Hook
const useOpenRouter = (config: OpenRouterConfig) => {
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [totalTokens, setTotalTokens] = useState(0);
  const [totalCost, setTotalCost] = useState(0);

  const generateCompletion = useCallback(async (
    messages: Message[],
    context?: CodeContext
  ): Promise<string> => {
    if (!config.apiKey) {
      throw new Error('OpenRouter API key is required');
    }

    setIsLoading(true);
    setError(null);
    
    try {
      const systemMessage = context ? buildSystemPrompt(context) : '';
      const requestMessages = [
        ...(systemMessage ? [{ role: 'system' as const, content: systemMessage }] : []),
        ...messages.map(msg => ({
          role: msg.role,
          content: msg.content
        }))
      ];
    
      const response = await fetch(`${config.baseUrl}/chat/completions`, {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${config.apiKey}`,
          'Content-Type': 'application/json',
          'HTTP-Referer': window.location.origin,
          'X-Title': 'QuickBuild Framework'
        },
        body: JSON.stringify({
          model: config.model,
          messages: requestMessages,
          temperature: config.temperature,
          max_tokens: config.maxTokens,
          top_p: config.topP,
          stream: false
        })
      });
    
      if (!response.ok) {
        const errorData = await response.json();
        throw new Error(errorData.error?.message || `HTTP ${response.status}: ${response.statusText}`);
      }
    
      const data: OpenRouterResponse = await response.json();
    
      // Update usage statistics
      if (data.usage) {
        setTotalTokens(prev => prev + data.usage.total_tokens);
        // Estimate cost (example rates - adjust based on actual OpenRouter pricing)
        const estimatedCost = (data.usage.total_tokens / 1000) * 0.002; // $0.002 per 1K tokens
        setTotalCost(prev => prev + estimatedCost);
      }
    
      return data.choices[0]?.message?.content || '';
    } catch (err) {
      const errorMessage = err instanceof Error ? err.message : 'Unknown error occurred';
      setError(errorMessage);
      throw new Error(errorMessage);
    } finally {
      setIsLoading(false);
    }

  }, [config]);

  const generateCode = useCallback(async (
    prompt: string,
    context: CodeContext
  ): Promise<string> => {
    const messages: Message[] = [{
      id: Date.now().toString(),
      role: 'user',
      content: prompt,
      timestamp: new Date()
    }];

    return generateCompletion(messages, context);

  }, [generateCompletion]);

  const explainCode = useCallback(async (
    code: string,
    language: string
  ): Promise<string> => {
    const messages: Message[] = [{
      id: Date.now().toString(),
      role: 'user',
      content: `Please explain this ${language} code:\n\n\`\`\`${language}\n${code}\n\`\`\``,
      timestamp: new Date()
    }];

    return generateCompletion(messages);

  }, [generateCompletion]);

  const optimizeCode = useCallback(async (
    code: string,
    language: string,
    context?: CodeContext
  ): Promise<string> => {
    const messages: Message[] = [{
      id: Date.now().toString(),
      role: 'user',
      content: `Please optimize this ${language} code for performance and best practices:\n\n\`\`\`${language}\n${code}\n\`\`\``,
      timestamp: new Date()
    }];

    return generateCompletion(messages, context);

  }, [generateCompletion]);

  return {
    generateCompletion,
    generateCode,
    explainCode,
    optimizeCode,
    isLoading,
    error,
    totalTokens,
    totalCost
  };
};

// Helper function to build system prompt
const buildSystemPrompt = (context: CodeContext): string => {
  return `You are an expert ${context.framework || context.language} developer assistant integrated into QuickBuild Framework.

Current Context:

- File: ${context.fileName}
- Language: ${context.language}
- Framework: ${context.framework}
- Dependencies: ${context.dependencies.join(', ')}

Guidelines:

1. Generate clean, production-ready code
2. Follow ${context.framework || context.language} best practices
3. Use TypeScript when possible
4. Include proper error handling
5. Optimize for performance
6. Follow accessibility guidelines
7. Return only the code without explanations unless asked

Current file content for context:
\`\`\`${context.language}
${context.fileContent}
\`\`\``;
};

// Available Models Configuration
const AVAILABLE_MODELS = [
  { id: 'anthropic/claude-3.5-sonnet', name: 'Claude 3.5 Sonnet', provider: 'Anthropic' },
  { id: 'openai/gpt-4', name: 'GPT-4', provider: 'OpenAI' },
  { id: 'openai/gpt-3.5-turbo', name: 'GPT-3.5 Turbo', provider: 'OpenAI' },
  { id: 'meta-llama/llama-2-70b-chat', name: 'Llama 2 70B', provider: 'Meta' },
  { id: 'google/palm-2-chat-bison', name: 'PaLM 2 Chat', provider: 'Google' },
  { id: 'cohere/command', name: 'Command', provider: 'Cohere' }
];

// Main OpenRouter Integration Component
const OpenRouterIntegration: React.FC = () => {
  const [config, setConfig] = useState<OpenRouterConfig>({
    apiKey: '',
    model: 'anthropic/claude-3.5-sonnet',
    baseUrl: 'https://openrouter.ai/api/v1',
    temperature: 0.7,
    maxTokens: 4000,
    topP: 1.0
  });

  const [messages, setMessages] = useState<Message[]>([]);
  const [inputValue, setInputValue] = useState('');
  const [showSettings, setShowSettings] = useState(false);
  const [codeContext, setCodeContext] = useState<CodeContext>({
    fileName: 'example.tsx',
    fileContent: '',
    language: 'typescript',
    framework: 'react',
    dependencies: ['react', '@types/react', 'tailwindcss']
  });

  const messagesEndRef = useRef<HTMLDivElement>(null);
  const { 
    generateCompletion, 
    generateCode, 
    explainCode, 
    optimizeCode, 
    isLoading, 
    error, 
    totalTokens, 
    totalCost 
  } = useOpenRouter(config);

  // Auto-scroll to bottom when new messages arrive
  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages]);

  // Load saved configuration
  useEffect(() => {
    const savedConfig = localStorage.getItem('openrouter-config');
    if (savedConfig) {
      try {
        const parsed = JSON.parse(savedConfig);
        setConfig(prev => ({ ...prev, ...parsed }));
      } catch (err) {
        console.error('Failed to load saved config:', err);
      }
    }
  }, []);

  // Save configuration to localStorage
  const saveConfig = useCallback((newConfig: Partial<OpenRouterConfig>) => {
    const updatedConfig = { ...config, ...newConfig };
    setConfig(updatedConfig);
    localStorage.setItem('openrouter-config', JSON.stringify(updatedConfig));
  }, [config]);

  const handleSendMessage = async () => {
    if (!inputValue.trim() || isLoading) return;

    const userMessage: Message = {
      id: Date.now().toString(),
      role: 'user',
      content: inputValue,
      timestamp: new Date()
    };
    
    setMessages(prev => [...prev, userMessage]);
    setInputValue('');
    
    try {
      const response = await generateCompletion([userMessage], codeContext);
    
      const assistantMessage: Message = {
        id: (Date.now() + 1).toString(),
        role: 'assistant',
        content: response,
        timestamp: new Date()
      };
    
      setMessages(prev => [...prev, assistantMessage]);
    } catch (err) {
      console.error('Failed to generate response:', err);
    }

  };

  const handleQuickAction = async (action: string) => {
    if (!codeContext.fileContent.trim()) {
      alert('Please add some code content first');
      return;
    }

    try {
      let response = '';
      switch (action) {
        case 'explain':
          response = await explainCode(codeContext.fileContent, codeContext.language);
          break;
        case 'optimize':
          response = await optimizeCode(codeContext.fileContent, codeContext.language, codeContext);
          break;
        case 'generate':
          response = await generateCode('Create a simple component based on this code', codeContext);
          break;
      }
    
      const assistantMessage: Message = {
        id: Date.now().toString(),
        role: 'assistant',
        content: response,
        timestamp: new Date()
      };
    
      setMessages(prev => [...prev, assistantMessage]);
    } catch (err) {
      console.error(`Failed to ${action} code:`, err);
    }

  };

  const copyToClipboard = (text: string) => {
    navigator.clipboard.writeText(text);
  };

  return (
    <div className="w-full max-w-6xl mx-auto bg-white dark:bg-gray-900 rounded-lg shadow-lg overflow-hidden">
      {/* Header */}
      <div className="bg-gradient-to-r from-blue-600 to-purple-600 text-white p-4">
        <div className="flex items-center justify-between">
          <div className="flex items-center space-x-3">
            <Zap className="w-6 h-6" />
            <h2 className="text-xl font-bold">OpenRouter AI Integration</h2>
          </div>
          <div className="flex items-center space-x-2">
            <span className="text-sm opacity-90">Tokens: {totalTokens.toLocaleString()}</span>
            <span className="text-sm opacity-90">Cost: ${totalCost.toFixed(4)}</span>
            <button
              onClick={() => setShowSettings(!showSettings)}
              className="p-2 hover:bg-white/20 rounded-lg transition-colors"
            >
              <Settings className="w-5 h-5" />
            </button>
          </div>
        </div>
      </div>

      {/* Settings Panel */}
      {showSettings && (
        <div className="bg-gray-50 dark:bg-gray-800 p-4 border-b">
          <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
            <div>
              <label className="block text-sm font-medium mb-2">API Key</label>
              <div className="relative">
                <Key className="absolute left-3 top-3 w-4 h-4 text-gray-400" />
                <input
                  type="password"
                  value={config.apiKey}
                  onChange={(e) => saveConfig({ apiKey: e.target.value })}
                  placeholder="Enter your OpenRouter API key"
                  className="w-full pl-10 pr-4 py-2 border rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
                />
              </div>
            </div>
    
            <div>
              <label className="block text-sm font-medium mb-2">Model</label>
              <select
                value={config.model}
                onChange={(e) => saveConfig({ model: e.target.value })}
                className="w-full px-3 py-2 border rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
              >
                {AVAILABLE_MODELS.map(model => (
                  <option key={model.id} value={model.id}>
                    {model.name} ({model.provider})
                  </option>
                ))}
              </select>
            </div>
    
            <div>
              <label className="block text-sm font-medium mb-2">Temperature: {config.temperature}</label>
              <input
                type="range"
                min="0"
                max="2"
                step="0.1"
                value={config.temperature}
                onChange={(e) => saveConfig({ temperature: parseFloat(e.target.value) })}
                className="w-full"
              />
            </div>
    
            <div>
              <label className="block text-sm font-medium mb-2">Max Tokens</label>
              <input
                type="number"
                min="1"
                max="8000"
                value={config.maxTokens}
                onChange={(e) => saveConfig({ maxTokens: parseInt(e.target.value) })}
                className="w-full px-3 py-2 border rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
              />
            </div>
          </div>
        </div>
      )}
    
      <div className="flex h-96">
        {/* Code Context Panel */}
        <div className="w-1/3 bg-gray-50 dark:bg-gray-800 p-4 border-r">
          <h3 className="font-semibold mb-3">Code Context</h3>
    
          <div className="space-y-3">
            <div>
              <label className="block text-xs font-medium mb-1">File Name</label>
              <input
                type="text"
                value={codeContext.fileName}
                onChange={(e) => setCodeContext(prev => ({ ...prev, fileName: e.target.value }))}
                className="w-full px-2 py-1 text-sm border rounded focus:ring-1 focus:ring-blue-500"
              />
            </div>
    
            <div>
              <label className="block text-xs font-medium mb-1">Framework</label>
              <select
                value={codeContext.framework}
                onChange={(e) => setCodeContext(prev => ({ ...prev, framework: e.target.value }))}
                className="w-full px-2 py-1 text-sm border rounded focus:ring-1 focus:ring-blue-500"
              >
                <option value="react">React</option>
                <option value="vue">Vue</option>
                <option value="nextjs">Next.js</option>
                <option value="nuxt">Nuxt</option>
                <option value="svelte">Svelte</option>
              </select>
            </div>
    
            <div>
              <label className="block text-xs font-medium mb-1">Code Content</label>
              <textarea
                value={codeContext.fileContent}
                onChange={(e) => setCodeContext(prev => ({ ...prev, fileContent: e.target.value }))}
                placeholder="Paste your code here for context..."
                className="w-full px-2 py-1 text-xs border rounded focus:ring-1 focus:ring-blue-500 font-mono h-32 resize-none"
              />
            </div>
    
            <div className="flex flex-wrap gap-1">
              <button
                onClick={() => handleQuickAction('explain')}
                disabled={isLoading}
                className="px-2 py-1 text-xs bg-blue-100 text-blue-700 rounded hover:bg-blue-200 disabled:opacity-50"
              >
                Explain Code
              </button>
              <button
                onClick={() => handleQuickAction('optimize')}
                disabled={isLoading}
                className="px-2 py-1 text-xs bg-green-100 text-green-700 rounded hover:bg-green-200 disabled:opacity-50"
              >
                Optimize
              </button>
              <button
                onClick={() => handleQuickAction('generate')}
                disabled={isLoading}
                className="px-2 py-1 text-xs bg-purple-100 text-purple-700 rounded hover:bg-purple-200 disabled:opacity-50"
              >
                Generate
              </button>
            </div>
          </div>
        </div>
    
        {/* Chat Interface */}
        <div className="flex-1 flex flex-col">
          {/* Messages */}
          <div className="flex-1 overflow-y-auto p-4 space-y-4">
            {messages.length === 0 ? (
              <div className="text-center text-gray-500 mt-8">
                <MessageCircle className="w-12 h-12 mx-auto mb-4 opacity-50" />
                <p>Start a conversation with your AI assistant</p>
                <p className="text-sm mt-2">Ask questions, request code generation, or get explanations</p>
              </div>
            ) : (
              messages.map((message) => (
                <div
                  key={message.id}
                  className={`flex ${message.role === 'user' ? 'justify-end' : 'justify-start'}`}
                >
                  <div
                    className={`max-w-xs lg:max-w-md px-4 py-2 rounded-lg ${
                      message.role === 'user'
                        ? 'bg-blue-600 text-white'
                        : 'bg-gray-200 dark:bg-gray-700 text-gray-900 dark:text-gray-100'
                    }`}
                  >
                    {message.role === 'assistant' && message.content.includes('```') ? (
                      <div className="space-y-2">
                        {message.content.split('```').map((part, index) => (
                          <div key={index}>
                            {index % 2 === 0 ? (
                              <p className="whitespace-pre-wrap">{part}</p>
                            ) : (
                              <div className="relative">
                                <pre className="bg-gray-800 text-green-400 p-2 rounded text-xs overflow-x-auto">
                                  <code>{part}</code>
                                </pre>
                                <button
                                  onClick={() => copyToClipboard(part)}
                                  className="absolute top-1 right-1 p-1 text-gray-400 hover:text-white"
                                >
                                  <Copy className="w-3 h-3" />
                                </button>
                              </div>
                            )}
                          </div>
                        ))}
                      </div>
                    ) : (
                      <p className="whitespace-pre-wrap">{message.content}</p>
                    )}
                    <p className="text-xs opacity-70 mt-1">
                      {message.timestamp.toLocaleTimeString()}
                    </p>
                  </div>
                </div>
              ))
            )}
    
            {isLoading && (
              <div className="flex justify-start">
                <div className="bg-gray-200 dark:bg-gray-700 px-4 py-2 rounded-lg">
                  <div className="flex items-center space-x-2">
                    <RefreshCw className="w-4 h-4 animate-spin" />
                    <span>AI is thinking...</span>
                  </div>
                </div>
              </div>
            )}
    
            {error && (
              <div className="flex justify-center">
                <div className="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded flex items-center space-x-2">
                  <AlertCircle className="w-4 h-4" />
                  <span className="text-sm">{error}</span>
                </div>
              </div>
            )}
    
            <div ref={messagesEndRef} />
          </div>
    
          {/* Input */}
          <div className="border-t p-4">
            <div className="flex space-x-2">
              <input
                type="text"
                value={inputValue}
                onChange={(e) => setInputValue(e.target.value)}
                onKeyPress={(e) => e.key === 'Enter' && handleSendMessage()}
                placeholder="Ask anything about your code..."
                disabled={isLoading}
                className="flex-1 px-4 py-2 border rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent disabled:opacity-50"
              />
              <button
                onClick={handleSendMessage}
                disabled={isLoading || !inputValue.trim() || !config.apiKey}
                className="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 disabled:opacity-50 disabled:cursor-not-allowed flex items-center space-x-2"
              >
                <Send className="w-4 h-4" />
                <span>Send</span>
              </button>
            </div>
    
            {!config.apiKey && (
              <p className="text-xs text-orange-600 mt-2 flex items-center space-x-1">
                <AlertCircle className="w-3 h-3" />
                <span>Please configure your OpenRouter API key in settings</span>
              </p>
            )}
          </div>
        </div>
      </div>
    
      {/* Status Bar */}
      <div className="bg-gray-100 dark:bg-gray-800 px-4 py-2 text-xs text-gray-600 dark:text-gray-400 flex justify-between items-center">
        <div className="flex items-center space-x-4">
          <span>Model: {AVAILABLE_MODELS.find(m => m.id === config.model)?.name}</span>
          <span>Temp: {config.temperature}</span>
        </div>
        <div className="flex items-center space-x-2">
          {config.apiKey ? (
            <>
              <CheckCircle className="w-3 h-3 text-green-500" />
              <span>Connected</span>
            </>
          ) : (
            <>
              <AlertCircle className="w-3 h-3 text-orange-500" />
              <span>API Key Required</span>
            </>
          )}
        </div>
      </div>
    </div>

  );
};

export default OpenRouterIntegration;

https://claude.ai/public/artifacts/1840cc13-0f96-4085-bb33-7e74855279e3

# OpenRouter API Integration Guide - Complete Implementation

## Overview

This guide provides everything needed to integrate OpenRouter API into any application, with examples for React, Vue, Node.js, and Python implementations.

## Table of Contents

1. [Quick Start](#quick-start)
2. [API Configuration](#api-configuration)
3. [React Integration](#react-integration)
4. [Vue.js Integration](#vuejs-integration)
5. [Node.js Backend](#nodejs-backend)
6. [Python Implementation](#python-implementation)
7. [Security Best Practices](#security-best-practices)
8. [Error Handling](#error-handling)
9. [Performance Optimization](#performance-optimization)
10. [Testing Strategies](#testing-strategies)

## Quick Start

### 1. Get API Key

1. Sign up at [OpenRouter.ai](https://openrouter.ai)
2. Navigate to API Keys section
3. Generate new API key
4. Note your available models and rate limits

### 2. Install Dependencies

```bash
# For React/Node.js projects
npm install axios react-query # or your preferred HTTP client

# For Python projects
pip install requests python-dotenv
```

### 3. Basic Usage

```javascript
const response = await fetch('https://openrouter.ai/api/v1/chat/completions', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${process.env.OPENROUTER_API_KEY}`,
    'Content-Type': 'application/json',
    'HTTP-Referer': process.env.YOUR_SITE_URL,
    'X-Title': process.env.YOUR_APP_NAME
  },
  body: JSON.stringify({
    model: 'anthropic/claude-3.5-sonnet',
    messages: [
      { role: 'user', content: 'Hello, Claude!' }
    ]
  })
});
```

## API Configuration

### Environment Variables

```bash
# .env file
OPENROUTER_API_KEY=your_api_key_here
OPENROUTER_BASE_URL=https://openrouter.ai/api/v1
YOUR_SITE_URL=https://yourapp.com
YOUR_APP_NAME=Your App Name

# Optional: Model preferences
OPENROUTER_DEFAULT_MODEL=anthropic/claude-3.5-sonnet
OPENROUTER_MAX_TOKENS=4000
OPENROUTER_TEMPERATURE=0.7
```

### Model Selection Guide

```typescript
interface ModelInfo {
  id: string;
  name: string;
  provider: string;
  contextWindow: number;
  costPer1KTokens: number;
  capabilities: string[];
}

export const AVAILABLE_MODELS: ModelInfo[] = [
  {
    id: 'anthropic/claude-3.5-sonnet',
    name: 'Claude 3.5 Sonnet',
    provider: 'Anthropic',
    contextWindow: 200000,
    costPer1KTokens: 0.003,
    capabilities: ['coding', 'analysis', 'reasoning', 'creative-writing']
  },
  {
    id: 'openai/gpt-4',
    name: 'GPT-4',
    provider: 'OpenAI', 
    contextWindow: 8192,
    costPer1KTokens: 0.03,
    capabilities: ['coding', 'analysis', 'reasoning', 'creative-writing']
  },
  {
    id: 'openai/gpt-3.5-turbo',
    name: 'GPT-3.5 Turbo',
    provider: 'OpenAI',
    contextWindow: 16385,
    costPer1KTokens: 0.0015,
    capabilities: ['coding', 'conversation', 'analysis']
  },
  {
    id: 'meta-llama/llama-2-70b-chat',
    name: 'Llama 2 70B Chat',
    provider: 'Meta',
    contextWindow: 4096,
    costPer1KTokens: 0.0007,
    capabilities: ['conversation', 'reasoning', 'coding']
  }
];
```

## React Integration

### Hook Implementation

```typescript
// hooks/useOpenRouter.ts
import { useState, useCallback } from 'react';
import { useQuery, useMutation } from 'react-query';

interface OpenRouterConfig {
  apiKey: string;
  model: string;
  temperature?: number;
  maxTokens?: number;
  systemPrompt?: string;
}

interface Message {
  role: 'user' | 'assistant' | 'system';
  content: string;
}

export const useOpenRouter = (config: OpenRouterConfig) => {
  const [messages, setMessages] = useState<Message[]>([]);
  const [usage, setUsage] = useState({ tokens: 0, cost: 0 });

  const generateCompletion = useMutation(
    async (newMessages: Message[]) => {
      const requestMessages = config.systemPrompt 
        ? [{ role: 'system' as const, content: config.systemPrompt }, ...newMessages]
        : newMessages;

      const response = await fetch('https://openrouter.ai/api/v1/chat/completions', {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${config.apiKey}`,
          'Content-Type': 'application/json',
          'HTTP-Referer': window.location.origin,
          'X-Title': document.title
        },
        body: JSON.stringify({
          model: config.model,
          messages: requestMessages,
          temperature: config.temperature || 0.7,
          max_tokens: config.maxTokens || 4000
        })
      });

      if (!response.ok) {
        throw new Error(`OpenRouter API error: ${response.statusText}`);
      }

      const data = await response.json();

      // Update usage tracking
      if (data.usage) {
        setUsage(prev => ({
          tokens: prev.tokens + data.usage.total_tokens,
          cost: prev.cost + (data.usage.total_tokens * 0.002 / 1000) // Estimate
        }));
      }

      return data.choices[0]?.message?.content || '';
    },
    {
      onSuccess: (response, variables) => {
        setMessages(prev => [
          ...prev,
          ...variables,
          { role: 'assistant', content: response }
        ]);
      }
    }
  );

  const sendMessage = useCallback((content: string) => {
    const userMessage: Message = { role: 'user', content };
    const newMessages = [...messages, userMessage];
    generateCompletion.mutate(newMessages);
  }, [messages, generateCompletion]);

  const reset = useCallback(() => {
    setMessages([]);
    setUsage({ tokens: 0, cost: 0 });
  }, []);

  return {
    messages,
    sendMessage,
    reset,
    usage,
    isLoading: generateCompletion.isLoading,
    error: generateCompletion.error
  };
};
```

### Component Example

```tsx
// components/AIChat.tsx
import React, { useState } from 'react';
import { useOpenRouter } from '../hooks/useOpenRouter';

interface AIChatProps {
  apiKey: string;
  model?: string;
  systemPrompt?: string;
}

export const AIChat: React.FC<AIChatProps> = ({ 
  apiKey, 
  model = 'anthropic/claude-3.5-sonnet',
  systemPrompt 
}) => {
  const [input, setInput] = useState('');
  const { messages, sendMessage, isLoading, error, usage, reset } = useOpenRouter({
    apiKey,
    model,
    systemPrompt
  });

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    if (!input.trim()) return;

    sendMessage(input);
    setInput('');
  };

  return (
    <div className="max-w-2xl mx-auto p-4">
      <div className="mb-4 flex justify-between items-center">
        <h2 className="text-xl font-bold">AI Assistant</h2>
        <div className="text-sm text-gray-600">
          Tokens: {usage.tokens} | Cost: ${usage.cost.toFixed(4)}
        </div>
      </div>

      <div className="border rounded-lg p-4 h-96 overflow-y-auto mb-4">
        {messages.map((message, index) => (
          <div key={index} className={`mb-2 ${
            message.role === 'user' ? 'text-right' : 'text-left'
          }`}>
            <div className={`inline-block p-2 rounded-lg ${
              message.role === 'user' 
                ? 'bg-blue-500 text-white' 
                : 'bg-gray-100 text-gray-900'
            }`}>
              {message.content}
            </div>
          </div>
        ))}
        {isLoading && (
          <div className="text-center text-gray-500">
            AI is thinking...
          </div>
        )}
      </div>

      {error && (
        <div className="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded mb-4">
          Error: {error.message}
        </div>
      )}

      <form onSubmit={handleSubmit} className="flex gap-2">
        <input
          type="text"
          value={input}
          onChange={(e) => setInput(e.target.value)}
          placeholder="Type your message..."
          className="flex-1 border rounded-lg px-3 py-2"
          disabled={isLoading}
        />
        <button
          type="submit"
          disabled={isLoading || !input.trim()}
          className="bg-blue-500 text-white px-4 py-2 rounded-lg disabled:opacity-50"
        >
          Send
        </button>
        <button
          type="button"
          onClick={reset}
          className="bg-gray-500 text-white px-4 py-2 rounded-lg"
        >
          Reset
        </button>
      </form>
    </div>
  );
};
```

## Vue.js Integration

### Composable

```typescript
// composables/useOpenRouter.ts
import { ref, computed } from 'vue';

interface Message {
  role: 'user' | 'assistant' | 'system';
  content: string;
}

export function useOpenRouter(config: {
  apiKey: string;
  model: string;
  systemPrompt?: string;
}) {
  const messages = ref<Message[]>([]);
  const isLoading = ref(false);
  const error = ref<string | null>(null);
  const usage = ref({ tokens: 0, cost: 0 });

  const sendMessage = async (content: string) => {
    if (isLoading.value) return;

    isLoading.value = true;
    error.value = null;

    try {
      const userMessage: Message = { role: 'user', content };
      messages.value.push(userMessage);

      const requestMessages = config.systemPrompt 
        ? [{ role: 'system' as const, content: config.systemPrompt }, ...messages.value]
        : messages.value;

      const response = await fetch('https://openrouter.ai/api/v1/chat/completions', {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${config.apiKey}`,
          'Content-Type': 'application/json',
          'HTTP-Referer': window.location.origin,
          'X-Title': document.title
        },
        body: JSON.stringify({
          model: config.model,
          messages: requestMessages,
          temperature: 0.7,
          max_tokens: 4000
        })
      });

      if (!response.ok) {
        throw new Error(`API error: ${response.statusText}`);
      }

      const data = await response.json();
      const assistantMessage: Message = {
        role: 'assistant',
        content: data.choices[0]?.message?.content || ''
      };

      messages.value.push(assistantMessage);

      if (data.usage) {
        usage.value.tokens += data.usage.total_tokens;
        usage.value.cost += (data.usage.total_tokens * 0.002 / 1000);
      }
    } catch (err) {
      error.value = err instanceof Error ? err.message : 'Unknown error';
    } finally {
      isLoading.value = false;
    }
  };

  const reset = () => {
    messages.value = [];
    usage.value = { tokens: 0, cost: 0 };
    error.value = null;
  };

  return {
    messages: computed(() => messages.value),
    isLoading: computed(() => isLoading.value),
    error: computed(() => error.value),
    usage: computed(() => usage.value),
    sendMessage,
    reset
  };
}
```

## Node.js Backend

### Service Class

```typescript
// services/OpenRouterService.ts
import fetch from 'node-fetch';

interface OpenRouterConfig {
  apiKey: string;
  baseUrl?: string;
  defaultModel?: string;
  timeout?: number;
}

interface ChatMessage {
  role: 'user' | 'assistant' | 'system';
  content: string;
}

interface GenerationOptions {
  model?: string;
  temperature?: number;
  maxTokens?: number;
  systemPrompt?: string;
  timeout?: number;
}

export class OpenRouterService {
  private config: Required<OpenRouterConfig>;

  constructor(config: OpenRouterConfig) {
    this.config = {
      baseUrl: 'https://openrouter.ai/api/v1',
      defaultModel: 'anthropic/claude-3.5-sonnet',
      timeout: 30000,
      ...config
    };
  }

  async generateCompletion(
    messages: ChatMessage[],
    options: GenerationOptions = {}
  ): Promise<{
    content: string;
    usage: { prompt_tokens: number; completion_tokens: number; total_tokens: number };
    model: string;
  }> {
    const requestMessages = options.systemPrompt 
      ? [{ role: 'system' as const, content: options.systemPrompt }, ...messages]
      : messages;

    const requestBody = {
      model: options.model || this.config.defaultModel,
      messages: requestMessages,
      temperature: options.temperature || 0.7,
      max_tokens: options.maxTokens || 4000
    };

    const controller = new AbortController();
    const timeoutId = setTimeout(
      () => controller.abort(),
      options.timeout || this.config.timeout
    );

    try {
      const response = await fetch(`${this.config.baseUrl}/chat/completions`, {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${this.config.apiKey}`,
          'Content-Type': 'application/json',
          'HTTP-Referer': process.env.YOUR_SITE_URL || 'https://localhost',
          'X-Title': process.env.YOUR_APP_NAME || 'OpenRouter Integration'
        },
        body: JSON.stringify(requestBody),
        signal: controller.signal
      });

      if (!response.ok) {
        const errorData = await response.json().catch(() => ({}));
        throw new Error(`OpenRouter API error: ${response.status} - ${errorData.error?.message || response.statusText}`);
      }

      const data = await response.json();

      return {
        content: data.choices[0]?.message?.content || '',
        usage: data.usage || { prompt_tokens: 0, completion_tokens: 0, total_tokens: 0 },
        model: data.model
      };
    } catch (error) {
      if (error.name === 'AbortError') {
        throw new Error('Request timeout');
      }
      throw error;
    } finally {
      clearTimeout(timeoutId);
    }
  }

  async generateCode(
    prompt: string,
    context: {
      language: string;
      framework?: string;
      existingCode?: string;
    },
    options: GenerationOptions = {}
  ): Promise<string> {
    const systemPrompt = `You are an expert ${context.language} developer. Generate clean, production-ready code based on the user's request.

Context:
- Language: ${context.language}
- Framework: ${context.framework || 'None'}
${context.existingCode ? `- Existing code:\n\`\`\`${context.language}\n${context.existingCode}\n\`\`\`` : ''}

Guidelines:
1. Write clean, readable code
2. Follow best practices for ${context.language}
3. Include proper error handling
4. Add helpful comments
5. Return only the code without explanations`;

    const messages: ChatMessage[] = [
      { role: 'user', content: prompt }
    ];

    const result = await this.generateCompletion(messages, {
      ...options,
      systemPrompt
    });

    return result.content;
  }

  async explainCode(code: string, language: string): Promise<string> {
    const messages: ChatMessage[] = [
      {
        role: 'user',
        content: `Please explain this ${language} code:\n\n\`\`\`${language}\n${code}\n\`\`\``
      }
    ];

    const result = await this.generateCompletion(messages);
    return result.content;
  }
}
```

### Express.js API Routes

```typescript
// routes/ai.ts
import express from 'express';
import { OpenRouterService } from '../services/OpenRouterService';

const router = express.Router();
const openRouter = new OpenRouterService({
  apiKey: process.env.OPENROUTER_API_KEY!
});

router.post('/generate', async (req, res) => {
  try {
    const { messages, model, temperature, maxTokens } = req.body;

    const result = await openRouter.generateCompletion(messages, {
      model,
      temperature,
      maxTokens
    });

    res.json({
      success: true,
      data: result
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
});

router.post('/generate-code', async (req, res) => {
  try {
    const { prompt, context } = req.body;

    const code = await openRouter.generateCode(prompt, context);

    res.json({
      success: true,
      data: { code }
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
});

router.post('/explain-code', async (req, res) => {
  try {
    const { code, language } = req.body;

    const explanation = await openRouter.explainCode(code, language);

    res.json({
      success: true,
      data: { explanation }
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
});

export default router;
```

## Python Implementation

### Service Class

```python
# openrouter_service.py
import os
import json
import time
import requests
from typing import List, Dict, Optional, Union
from dataclasses import dataclass

@dataclass
class Message:
    role: str  # 'user', 'assistant', or 'system'
    content: str

@dataclass
class GenerationOptions:
    model: Optional[str] = None
    temperature: Optional[float] = 0.7
    max_tokens: Optional[int] = 4000
    system_prompt: Optional[str] = None
    timeout: Optional[int] = 30

class OpenRouterService:
    def __init__(self, api_key: str, base_url: str = "https://openrouter.ai/api/v1"):
        self.api_key = api_key
        self.base_url = base_url
        self.default_model = "anthropic/claude-3.5-sonnet"
        self.session = requests.Session()
        self.session.headers.update({
            'Authorization': f'Bearer {api_key}',
            'Content-Type': 'application/json',
            'HTTP-Referer': os.getenv('YOUR_SITE_URL', 'https://localhost'),
            'X-Title': os.getenv('YOUR_APP_NAME', 'OpenRouter Python Client')
        })

    def generate_completion(
        self, 
        messages: List[Message], 
        options: GenerationOptions = GenerationOptions()
    ) -> Dict:
        # Prepare messages
        request_messages = []
        if options.system_prompt:
            request_messages.append({"role": "system", "content": options.system_prompt})

        for msg in messages:
            request_messages.append({"role": msg.role, "content": msg.content})

        # Prepare request
        request_body = {
            "model": options.model or self.default_model,
            "messages": request_messages,
            "temperature": options.temperature,
            "max_tokens": options.max_tokens
        }

        try:
            response = self.session.post(
                f"{self.base_url}/chat/completions",
                json=request_body,
                timeout=options.timeout
            )
            response.raise_for_status()

            data = response.json()
            return {
                "content": data["choices"][0]["message"]["content"],
                "usage": data.get("usage", {}),
                "model": data.get("model", "")
            }
        except requests.exceptions.RequestException as e:
            raise Exception(f"OpenRouter API error: {str(e)}")

    def generate_code(
        self, 
        prompt: str, 
        context: Dict[str, str],
        options: GenerationOptions = GenerationOptions()
    ) -> str:
        language = context.get("language", "python")
        framework = context.get("framework", "")
        existing_code = context.get("existing_code", "")

        system_prompt = f"""You are an expert {language} developer. Generate clean, production-ready code based on the user's request.

Context:
- Language: {language}
- Framework: {framework}
{f"- Existing code:\n```{language}\n{existing_code}\n```" if existing_code else ""}

Guidelines:
1. Write clean, readable code
2. Follow best practices for {language}
3. Include proper error handling
4. Add helpful comments
5. Return only the code without explanations"""

        messages = [Message(role="user", content=prompt)]
        options.system_prompt = system_prompt

        result = self.generate_completion(messages, options)
        return result["content"]

    def explain_code(self, code: str, language: str) -> str:
        messages = [
            Message(
                role="user", 
                content=f"Please explain this {language} code:\n\n```{language}\n{code}\n```"
            )
        ]

        result = self.generate_completion(messages)
        return result["content"]

# Usage example
if __name__ == "__main__":
    from dotenv import load_dotenv
    load_dotenv()

    service = OpenRouterService(os.getenv("OPENROUTER_API_KEY"))

    # Generate code
    code = service.generate_code(
        "Create a function to calculate fibonacci numbers",
        {"language": "python"}
    )
    print("Generated code:", code)

    # Explain code
    explanation = service.explain_code(code, "python")
    print("Explanation:", explanation)
```

### Flask API

```python
# app.py
from flask import Flask, request, jsonify
from openrouter_service import OpenRouterService, Message, GenerationOptions
import os

app = Flask(__name__)
openrouter = OpenRouterService(os.getenv("OPENROUTER_API_KEY"))

@app.route('/api/generate', methods=['POST'])
def generate():
    try:
        data = request.json
        messages = [Message(**msg) for msg in data['messages']]
        options = GenerationOptions(**data.get('options', {}))

        result = openrouter.generate_completion(messages, options)
        return jsonify({"success": True, "data": result})
    except Exception as e:
        return jsonify({"success": False, "error": str(e)}), 500

@app.route('/api/generate-code', methods=['POST'])
def generate_code():
    try:
        data = request.json
        code = openrouter.generate_code(
            data['prompt'], 
            data['context'],
            GenerationOptions(**data.get('options', {}))
        )
        return jsonify({"success": True, "data": {"code": code}})
    except Exception as e:
        return jsonify({"success": False, "error": str(e)}), 500

@app.route('/api/explain-code', methods=['POST'])
def explain_code():
    try:
        data = request.json
        explanation = openrouter.explain_code(data['code'], data['language'])
        return jsonify({"success": True, "data": {"explanation": explanation}})
    except Exception as e:
        return jsonify({"success": False, "error": str(e)}), 500

if __name__ == '__main__':
    app.run(debug=True)
```

## Security Best Practices

### API Key Management

```typescript
// utils/secureStorage.ts (Electron)
import { safeStorage } from 'electron';

export class SecureAPIKeyManager {
  private static readonly KEY_PREFIX = 'openrouter_';

  static async storeAPIKey(service: string, key: string): Promise<void> {
    if (!safeStorage.isEncryptionAvailable()) {
      throw new Error('Encryption not available on this platform');
    }

    const encrypted = safeStorage.encryptString(key);
    localStorage.setItem(`${this.KEY_PREFIX}${service}`, encrypted.toString('base64'));
  }

  static async getAPIKey(service: string): Promise<string | null> {
    const stored = localStorage.getItem(`${this.KEY_PREFIX}${service}`);
    if (!stored) return null;

    try {
      const encrypted = Buffer.from(stored, 'base64');
      return safeStorage.decryptString(encrypted);
    } catch (error) {
      console.error('Failed to decrypt API key:', error);
      return null;
    }
  }

  static async removeAPIKey(service: string): Promise<void> {
    localStorage.removeItem(`${this.KEY_PREFIX}${service}`);
  }
}
```

### Request Validation

```typescript
// utils/validation.ts
import { z } from 'zod';

export const MessageSchema = z.object({
  role: z.enum(['user', 'assistant', 'system']),
  content: z.string().min(1).max(10000)
});

export const GenerationOptionsSchema = z.object({
  model: z.string().optional(),
  temperature: z.number().min(0).max(2).optional(),
  maxTokens: z.number().min(1).max(8000).optional(),
  systemPrompt: z.string().max(5000).optional()
});

export const validateMessages = (messages: unknown[]) => {
  return z.array(MessageSchema).parse(messages);
};

export const validateOptions = (options: unknown) => {
  return GenerationOptionsSchema.parse(options);
};
```

### Rate Limiting

```typescript
// utils/rateLimiter.ts
interface RateLimitConfig {
  maxRequests: number;
  windowMs: number;
}

export class RateLimiter {
  private requests: number[] = [];

  constructor(private config: RateLimitConfig) {}

  canMakeRequest(): boolean {
    const now = Date.now();
    const windowStart = now - this.config.windowMs;

    // Remove old requests
    this.requests = this.requests.filter(time => time > windowStart);

    return this.requests.length < this.config.maxRequests;
  }

  recordRequest(): void {
    this.requests.push(Date.now());
  }

  getTimeUntilReset(): number {
    if (this.requests.length === 0) return 0;

    const oldestRequest = Math.min(...this.requests);
    const resetTime = oldestRequest + this.config.windowMs;

    return Math.max(0, resetTime - Date.now());
  }
}

// Usage
const rateLimiter = new RateLimiter({ maxRequests: 10, windowMs: 60000 }); // 10 requests per minute
```

## Error Handling

### Comprehensive Error Types

```typescript
// types/errors.ts
export enum OpenRouterErrorType {
  AUTHENTICATION = 'authentication',
  RATE_LIMIT = 'rate_limit',
  INVALID_MODEL = 'invalid_model',
  NETWORK = 'network',
  TIMEOUT = 'timeout',
  INVALID_REQUEST = 'invalid_request',
  SERVER_ERROR = 'server_error',
  UNKNOWN = 'unknown'
}

export class OpenRouterError extends Error {
  constructor(
    public type: OpenRouterErrorType,
    message: string,
    public statusCode?: number,
    public retryAfter?: number
  ) {
    super(message);
    this.name = 'OpenRouterError';
  }

  static fromResponse(response: Response, data?: any): OpenRouterError {
    const message = data?.error?.message || response.statusText;

    switch (response.status) {
      case 401:
        return new OpenRouterError(OpenRouterErrorType.AUTHENTICATION, message, 401);
      case 429:
        const retryAfter = parseInt(response.headers.get('retry-after') || '60');
        return new OpenRouterError(OpenRouterErrorType.RATE_LIMIT, message, 429, retryAfter);
      case 400:
        return new OpenRouterError(OpenRouterErrorType.INVALID_REQUEST, message, 400);
      case 500:
      case 502:
      case 503:
        return new OpenRouterError(OpenRouterErrorType.SERVER_ERROR, message, response.status);
      default:
        return new OpenRouterError(OpenRouterErrorType.UNKNOWN, message, response.status);
    }
  }
}
```

### Retry Logic

```typescript
// utils/retry.ts
interface RetryOptions {
  maxAttempts: number;
  baseDelay: number;
  maxDelay: number;
  backoffFactor: number;
}

export async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  options: RetryOptions = {
    maxAttempts: 3,
    baseDelay: 1000,
    maxDelay: 10000,
    backoffFactor: 2
  }
): Promise<T> {
  let lastError: Error;

  for (let attempt = 1; attempt <= options.maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;

      if (attempt === options.maxAttempts) {
        break;
      }

      // Don't retry on certain errors
      if (error instanceof OpenRouterError) {
        if ([
          OpenRouterErrorType.AUTHENTICATION,
          OpenRouterErrorType.INVALID_REQUEST,
          OpenRouterErrorType.INVALID_MODEL
        ].includes(error.type)) {
          throw error;
        }

        // Use retry-after header for rate limits
        if (error.type === OpenRouterErrorType.RATE_LIMIT && error.retryAfter) {
          await delay(error.retryAfter * 1000);
          continue;
        }
      }

      // Calculate backoff delay
      const delay = Math.min(
        options.baseDelay * Math.pow(options.backoffFactor, attempt - 1),
        options.maxDelay
      );

      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }

  throw lastError!;
}

function delay(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms));
}
```

## Performance Optimization

### Response Caching

```typescript
// utils/cache.ts
interface CacheEntry<T> {
  data: T;
  timestamp: number;
  ttl: number;
}

export class ResponseCache<T> {
  private cache = new Map<string, CacheEntry<T>>();

  constructor(private defaultTTL: number = 300000) {} // 5 minutes default

  set(key: string, data: T, ttl?: number): void {
    this.cache.set(key, {
      data,
      timestamp: Date.now(),
      ttl: ttl || this.defaultTTL
    });
  }

  get(key: string): T | null {
    const entry = this.cache.get(key);
    if (!entry) return null;

    if (Date.now() - entry.timestamp > entry.ttl) {
      this.cache.delete(key);
      return null;
    }

    return entry.data;
  }

  clear(): void {
    this.cache.clear();
  }

  // Generate cache key from messages
  static generateKey(messages: Message[], options: any = {}): string {
    const content = JSON.stringify({ messages, options });
    return btoa(content).slice(0, 64); // Simple hash
  }
}

// Usage in OpenRouter service
const responseCache = new ResponseCache();

export const generateCompletionWithCache = async (
  messages: Message[],
  options: GenerationOptions = {}
): Promise<string> => {
  const cacheKey = ResponseCache.generateKey(messages, options);
  const cached = responseCache.get(cacheKey);

  if (cached) {
    return cached;
  }

  const result = await generateCompletion(messages, options);
  responseCache.set(cacheKey, result.content);

  return result.content;
};
```

### Streaming Responses

```typescript
// utils/streaming.ts
export async function* streamCompletion(
  messages: Message[],
  config: OpenRouterConfig
): AsyncGenerator<string, void, unknown> {
  const response = await fetch('https://openrouter.ai/api/v1/chat/completions', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${config.apiKey}`,
      'Content-Type': 'application/json',
      'HTTP-Referer': window.location.origin,
      'X-Title': document.title
    },
    body: JSON.stringify({
      model: config.model,
      messages,
      stream: true
    })
  });

  if (!response.ok) {
    throw new Error(`OpenRouter API error: ${response.statusText}`);
  }

  const reader = response.body?.getReader();
  if (!reader) throw new Error('Failed to get response reader');

  const decoder = new TextDecoder();
  let buffer = '';

  try {
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      buffer += decoder.decode(value, { stream: true });
      const lines = buffer.split('\n');
      buffer = lines.pop() || '';

      for (const line of lines) {
        if (line.startsWith('data: ')) {
          const data = line.slice(6);
          if (data === '[DONE]') return;

          try {
            const parsed = JSON.parse(data);
            const content = parsed.choices[0]?.delta?.content;
            if (content) {
              yield content;
            }
          } catch (error) {
            console.warn('Failed to parse streaming response:', error);
          }
        }
      }
    }
  } finally {
    reader.releaseLock();
  }
}

// React hook for streaming
export const useStreamingCompletion = (config: OpenRouterConfig) => {
  const [content, setContent] = useState('');
  const [isStreaming, setIsStreaming] = useState(false);

  const startStream = async (messages: Message[]) => {
    setContent('');
    setIsStreaming(true);

    try {
      for await (const chunk of streamCompletion(messages, config)) {
        setContent(prev => prev + chunk);
      }
    } catch (error) {
      console.error('Streaming error:', error);
    } finally {
      setIsStreaming(false);
    }
  };

  return { content, isStreaming, startStream };
};
```

## Testing Strategies

### Unit Tests

```typescript
// __tests__/openrouter.test.ts
import { jest } from '@jest/globals';
import { OpenRouterService } from '../src/services/OpenRouterService';

// Mock fetch
global.fetch = jest.fn();

describe('OpenRouterService', () => {
  let service: OpenRouterService;

  beforeEach(() => {
    service = new OpenRouterService({
      apiKey: 'test-key',
      defaultModel: 'anthropic/claude-3.5-sonnet'
    });
    jest.clearAllMocks();
  });

  test('should generate completion successfully', async () => {
    const mockResponse = {
      choices: [{ message: { content: 'Test response' } }],
      usage: { total_tokens: 100 },
      model: 'anthropic/claude-3.5-sonnet'
    };

    (fetch as jest.MockedFunction<typeof fetch>).mockResolvedValueOnce({
      ok: true,
      json: () => Promise.resolve(mockResponse)
    } as Response);

    const result = await service.generateCompletion([
      { role: 'user', content: 'Hello' }
    ]);

    expect(result.content).toBe('Test response');
    expect(result.usage.total_tokens).toBe(100);
  });

  test('should handle API errors', async () => {
    (fetch as jest.MockedFunction<typeof fetch>).mockResolvedValueOnce({
      ok: false,
      status: 401,
      statusText: 'Unauthorized',
      json: () => Promise.resolve({ error: { message: 'Invalid API key' } })
    } as Response);

    await expect(
      service.generateCompletion([{ role: 'user', content: 'Hello' }])
    ).rejects.toThrow('OpenRouter API error: 401 - Invalid API key');
  });
});
```

### Integration Tests

```typescript
// __tests__/integration.test.ts
import { OpenRouterService } from '../src/services/OpenRouterService';

describe('OpenRouter Integration', () => {
  let service: OpenRouterService;

  beforeAll(() => {
    const apiKey = process.env.OPENROUTER_API_KEY;
    if (!apiKey) {
      throw new Error('OPENROUTER_API_KEY environment variable required for integration tests');
    }

    service = new OpenRouterService({ apiKey });
  });

  test('should generate code from prompt', async () => {
    const code = await service.generateCode(
      'Create a simple Hello World function',
      { language: 'javascript' }
    );

    expect(code).toContain('function');
    expect(code).toContain('Hello');
  }, 30000); // 30 second timeout

  test('should explain code correctly', async () => {
    const explanation = await service.explainCode(
      'function add(a, b) { return a + b; }',
      'javascript'
    );

    expect(explanation).toContain('function');
    expect(explanation).toContain('add');
  }, 30000);
});
```

This comprehensive integration guide provides everything needed to implement OpenRouter API across different platforms and frameworks, with proper security, error handling, and performance optimizations.
