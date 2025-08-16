# 创建你的第一个 MCP 服务器

在这一章中，我们将从零开始创建一个完整的 MCP 服务器。你将学会如何实现工具、资源和提示词，以及如何测试和部署你的服务器。

## 开发环境搭建

### 项目初始化

首先，让我们创建一个新的 Node.js 项目：

```bash
# 创建项目目录
mkdir my-mcp-server
cd my-mcp-server

# 初始化 npm 项目
npm init -y

# 安装 MCP SDK
npm install @modelcontextprotocol/sdk

# 安装开发依赖
npm install --save-dev typescript @types/node tsx
```

**为什么选择这些依赖？**
- `@modelcontextprotocol/sdk`: MCP 的官方 SDK，提供了构建服务器的所有必要工具
- `typescript`: 提供类型安全和更好的开发体验
- `@types/node`: Node.js 的 TypeScript 类型定义
- `tsx`: TypeScript 执行器，用于直接运行 TypeScript 代码

### 项目结构设计

创建以下目录结构：

```
my-mcp-server/
├── src/
│   ├── index.ts          # 主入口文件
│   ├── tools/           # 工具实现
│   │   ├── calculator.ts
│   │   └── weather.ts
│   ├── resources/       # 资源实现
│   │   └── config.ts
│   └── prompts/         # 提示词实现
│       └── assistant.ts
├── package.json
├── tsconfig.json
└── README.md
```

**项目结构解释：**
- `src/`: 包含所有源代码
- `tools/`: 实现各种工具功能
- `resources/`: 提供可访问的资源
- `prompts/`: 定义提示词模板

### TypeScript 配置

创建 `tsconfig.json`：

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "node",
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### 更新 package.json

添加启动脚本：

```json
{
  "name": "my-mcp-server",
  "version": "1.0.0",
  "type": "module",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "tsx src/index.ts"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^0.5.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "tsx": "^4.0.0",
    "typescript": "^5.0.0"
  }
}
```

## 创建基础服务器结构

### 主入口文件

创建 `src/index.ts`：

```typescript
#!/usr/bin/env node

/**
 * 我的第一个 MCP 服务器
 * 
 * 这个服务器演示了如何实现：
 * - 工具 (Tools): 计算器和天气查询
 * - 资源 (Resources): 配置信息
 * - 提示词 (Prompts): AI 助手模板
 */

import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
  ListResourcesRequestSchema,
  ReadResourceRequestSchema,
  ListPromptsRequestSchema,
  GetPromptRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';

// 导入我们的功能模块
import { setupCalculatorTool } from './tools/calculator.js';
import { setupWeatherTool } from './tools/weather.js';
import { setupConfigResource } from './resources/config.js';
import { setupAssistantPrompt } from './prompts/assistant.js';

/**
 * 创建和配置 MCP 服务器
 */
async function createServer() {
  // 创建服务器实例
  const server = new Server(
    {
      name: 'my-mcp-server',
      version: '1.0.0',
    },
    {
      capabilities: {
        // 声明服务器支持的功能
        tools: {},      // 支持工具
        resources: {},  // 支持资源
        prompts: {},    // 支持提示词
      },
    }
  );

  // 设置工具
  setupCalculatorTool(server);
  setupWeatherTool(server);

  // 设置资源
  setupConfigResource(server);

  // 设置提示词
  setupAssistantPrompt(server);

  return server;
}

/**
 * 启动服务器
 */
async function main() {
  const server = await createServer();
  
  // 使用标准输入输出传输（Claude Desktop 使用这种方式）
  const transport = new StdioServerTransport();
  
  // 连接服务器和传输
  await server.connect(transport);
  
  console.error('MCP 服务器已启动并等待连接...');
}

// 错误处理
process.on('SIGINT', async () => {
  console.error('服务器正在关闭...');
  process.exit(0);
});

process.on('uncaughtException', (error) => {
  console.error('未捕获的异常:', error);
  process.exit(1);
});

// 启动服务器
main().catch((error) => {
  console.error('服务器启动失败:', error);
  process.exit(1);
});
```

**代码解释：**

1. **服务器初始化**：创建 MCP 服务器实例，指定名称和版本
2. **能力声明**：告诉客户端这个服务器支持哪些功能
3. **功能模块化**：将不同类型的功能分离到不同的文件中
4. **传输层**：使用 StdioServerTransport 通过标准输入输出通信
5. **错误处理**：优雅地处理各种错误情况

## 实现工具 (Tools)

工具是 AI 可以调用的函数。让我们实现两个实用的工具。

### 计算器工具

创建 `src/tools/calculator.ts`：

```typescript
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';

/**
 * 计算器工具的实现
 * 
 * 这个工具提供基本的数学计算功能：
 * - 加法、减法、乘法、除法
 * - 错误处理和输入验证
 */

// 定义支持的运算类型
type Operation = 'add' | 'subtract' | 'multiply' | 'divide';

/**
 * 执行数学计算
 */
function calculate(operation: Operation, a: number, b: number): number {
  switch (operation) {
    case 'add':
      return a + b;
    case 'subtract':
      return a - b;
    case 'multiply':
      return a * b;
    case 'divide':
      if (b === 0) {
        throw new Error('除数不能为零');
      }
      return a / b;
    default:
      throw new Error(`不支持的运算: ${operation}`);
  }
}

/**
 * 设置计算器工具
 */
export function setupCalculatorTool(server: Server) {
  // 注册工具列表处理器
  server.setRequestHandler(ListToolsRequestSchema, async () => {
    return {
      tools: [
        {
          name: 'calculator',
          description: '执行基本的数学计算（加、减、乘、除）',
          inputSchema: {
            type: 'object',
            properties: {
              operation: {
                type: 'string',
                enum: ['add', 'subtract', 'multiply', 'divide'],
                description: '要执行的数学运算'
              },
              a: {
                type: 'number',
                description: '第一个数字'
              },
              b: {
                type: 'number',
                description: '第二个数字'
              }
            },
            required: ['operation', 'a', 'b']
          }
        }
      ]
    };
  });

  // 注册工具调用处理器
  server.setRequestHandler(CallToolRequestSchema, async (request) => {
    const { name, arguments: args } = request.params;

    if (name === 'calculator') {
      try {
        // 验证参数
        const { operation, a, b } = args as {
          operation: Operation;
          a: number;
          b: number;
        };

        // 执行计算
        const result = calculate(operation, a, b);

        return {
          content: [
            {
              type: 'text',
              text: `计算结果: ${a} ${getOperationSymbol(operation)} ${b} = ${result}`
            }
          ]
        };
      } catch (error) {
        // 返回错误信息
        return {
          content: [
            {
              type: 'text',
              text: `计算错误: ${error instanceof Error ? error.message : '未知错误'}`
            }
          ],
          isError: true
        };
      }
    }

    throw new Error(`未知工具: ${name}`);
  });
}

/**
 * 获取运算符号
 */
function getOperationSymbol(operation: Operation): string {
  const symbols = {
    add: '+',
    subtract: '-',
    multiply: '×',
    divide: '÷'
  };
  return symbols[operation];
}
```

**工具实现要点：**

1. **输入验证**：使用 JSON Schema 定义输入参数的类型和约束
2. **错误处理**：捕获和处理各种错误情况（如除零错误）
3. **类型安全**：使用 TypeScript 确保类型正确
4. **用户友好**：返回格式化的、易于理解的结果

### 天气查询工具

创建 `src/tools/weather.ts`：

```typescript
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';

/**
 * 天气查询工具
 * 
 * 注意：这是一个演示工具，返回模拟的天气数据
 * 在实际应用中，你可以集成真实的天气 API
 */

interface WeatherData {
  location: string;
  temperature: number;
  condition: string;
  humidity: number;
  windSpeed: number;
}

/**
 * 模拟天气数据生成器
 * 在实际应用中，这里会调用真实的天气 API
 */
function getWeatherData(location: string): WeatherData {
  // 模拟不同城市的天气数据
  const weatherConditions = ['晴朗', '多云', '小雨', '阴天', '雾霾'];
  const randomCondition = weatherConditions[Math.floor(Math.random() * weatherConditions.length)];
  
  return {
    location,
    temperature: Math.round(Math.random() * 35 + 5), // 5-40度
    condition: randomCondition,
    humidity: Math.round(Math.random() * 100), // 0-100%
    windSpeed: Math.round(Math.random() * 20 + 1) // 1-20 km/h
  };
}

/**
 * 设置天气查询工具
 */
export function setupWeatherTool(server: Server) {
  // 这里我们需要合并工具列表，而不是覆盖
  server.setRequestHandler(ListToolsRequestSchema, async () => {
    return {
      tools: [
        {
          name: 'weather',
          description: '查询指定地点的当前天气信息',
          inputSchema: {
            type: 'object',
            properties: {
              location: {
                type: 'string',
                description: '要查询天气的城市或地点名称'
              }
            },
            required: ['location']
          }
        }
      ]
    };
  });

  server.setRequestHandler(CallToolRequestSchema, async (request) => {
    const { name, arguments: args } = request.params;

    if (name === 'weather') {
      try {
        const { location } = args as { location: string };

        // 获取天气数据
        const weather = getWeatherData(location);

        // 格式化天气报告
        const weatherReport = `
📍 地点: ${weather.location}
🌡️ 温度: ${weather.temperature}°C
☁️ 天气: ${weather.condition}
💧 湿度: ${weather.humidity}%
💨 风速: ${weather.windSpeed} km/h

${getWeatherAdvice(weather)}
        `.trim();

        return {
          content: [
            {
              type: 'text',
              text: weatherReport
            }
          ]
        };
      } catch (error) {
        return {
          content: [
            {
              type: 'text',
              text: `天气查询失败: ${error instanceof Error ? error.message : '未知错误'}`
            }
          ],
          isError: true
        };
      }
    }

    throw new Error(`未知工具: ${name}`);
  });
}

/**
 * 根据天气条件给出建议
 */
function getWeatherAdvice(weather: WeatherData): string {
  const { temperature, condition } = weather;

  let advice = '';

  // 温度建议
  if (temperature < 10) {
    advice += '🧥 建议穿厚外套保暖。';
  } else if (temperature < 20) {
    advice += '👕 建议穿长袖或薄外套。';
  } else if (temperature < 30) {
    advice += '👔 天气宜人，适合户外活动。';
  } else {
    advice += '🌞 天气炎热，注意防暑降温。';
  }

  // 天气条件建议
  if (condition.includes('雨')) {
    advice += ' ☂️ 记得带雨伞。';
  } else if (condition.includes('雾')) {
    advice += ' 🚗 出行注意安全，能见度较低。';
  }

  return advice;
}
```

**重要问题解决：**

目前我们有一个问题：两个工具都注册了 `ListToolsRequestSchema` 处理器，后注册的会覆盖前面的。我们需要修改架构来合并所有工具。

让我们更新 `src/index.ts` 来正确处理多个工具：

```typescript
// 在 src/index.ts 中，我们需要收集所有工具，然后统一注册

import { ToolDefinition } from '@modelcontextprotocol/sdk/types.js';

// 存储所有工具定义
const tools: ToolDefinition[] = [];

// 修改工具设置函数，让它们返回工具定义而不是直接注册
export function registerTool(tool: ToolDefinition) {
  tools.push(tool);
}

// 在服务器设置中统一注册所有工具
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return { tools };
});
```

让我们修改工具文件以使用新的注册方式。

## 实现资源 (Resources)

资源提供只读的信息访问。让我们创建一个配置资源。

创建 `src/resources/config.ts`：

```typescript
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import {
  ListResourcesRequestSchema,
  ReadResourceRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';

/**
 * 配置资源实现
 * 
 * 这个资源提供应用程序的配置信息，让 AI 能够了解：
 * - 服务器设置
 * - 功能开关
 * - 用户偏好
 */

interface AppConfig {
  server: {
    name: string;
    version: string;
    environment: string;
  };
  features: {
    calculatorEnabled: boolean;
    weatherEnabled: boolean;
    advancedMath: boolean;
  };
  preferences: {
    language: string;
    timezone: string;
    dateFormat: string;
  };
}

/**
 * 获取应用配置
 */
function getAppConfig(): AppConfig {
  return {
    server: {
      name: 'my-mcp-server',
      version: '1.0.0',
      environment: process.env.NODE_ENV || 'development'
    },
    features: {
      calculatorEnabled: true,
      weatherEnabled: true,
      advancedMath: false // 未来功能
    },
    preferences: {
      language: 'zh-CN',
      timezone: 'Asia/Shanghai',
      dateFormat: 'YYYY-MM-DD'
    }
  };
}

/**
 * 设置配置资源
 */
export function setupConfigResource(server: Server) {
  // 注册资源列表处理器
  server.setRequestHandler(ListResourcesRequestSchema, async () => {
    return {
      resources: [
        {
          uri: 'config://app',
          name: '应用配置',
          description: '当前应用程序的配置信息和设置',
          mimeType: 'application/json'
        }
      ]
    };
  });

  // 注册资源读取处理器
  server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
    const { uri } = request.params;

    if (uri === 'config://app') {
      const config = getAppConfig();
      
      return {
        contents: [
          {
            uri,
            mimeType: 'application/json',
            text: JSON.stringify(config, null, 2)
          }
        ]
      };
    }

    throw new Error(`未知资源: ${uri}`);
  });
}
```

**资源实现要点：**

1. **URI 方案**：使用自定义 URI 方案 `config://` 来标识资源
2. **MIME 类型**：正确设置 MIME 类型，帮助客户端理解内容格式
3. **结构化数据**：提供清晰、有层次的配置信息
4. **动态内容**：配置可以基于环境变量或其他条件动态生成

## 实现提示词 (Prompts)

提示词是预定义的模板，帮助 AI 处理特定类型的任务。

创建 `src/prompts/assistant.ts`：

```typescript
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import {
  ListPromptsRequestSchema,
  GetPromptRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';

/**
 * AI 助手提示词模板
 * 
 * 这些提示词帮助 AI 在不同场景下提供更好的服务：
 * - 代码审查
 * - 学习辅导
 * - 问题解决
 */

interface PromptTemplate {
  name: string;
  description: string;
  arguments?: Array<{
    name: string;
    description: string;
    required: boolean;
  }>;
  template: string;
}

/**
 * 提示词模板定义
 */
const promptTemplates: Record<string, PromptTemplate> = {
  'code-review': {
    name: 'code-review',
    description: '代码审查助手，帮助分析代码质量和提出改进建议',
    arguments: [
      {
        name: 'code',
        description: '要审查的代码内容',
        required: true
      },
      {
        name: 'language',
        description: '编程语言（如：JavaScript, Python, TypeScript）',
        required: false
      }
    ],
    template: `
作为一个经验丰富的代码审查专家，请仔细分析以下代码：

编程语言: {{language || "未指定"}}

代码内容:
\`\`\`{{language || ""}}
{{code}}
\`\`\`

请从以下几个方面进行审查：

1. **代码质量**
   - 代码可读性和清晰度
   - 变量和函数命名是否合理
   - 代码结构是否良好

2. **最佳实践**
   - 是否遵循该语言的最佳实践
   - 错误处理是否充分
   - 性能考虑

3. **潜在问题**
   - 安全漏洞
   - 边界情况处理
   - 内存泄漏或性能问题

4. **改进建议**
   - 具体的改进建议
   - 替代实现方案
   - 优化思路

请提供详细、具体、可操作的反馈意见。
    `.trim()
  },

  'learning-tutor': {
    name: 'learning-tutor',
    description: '学习辅导老师，帮助解释概念和指导学习',
    arguments: [
      {
        name: 'topic',
        description: '要学习的主题或概念',
        required: true
      },
      {
        name: 'level',
        description: '学习者水平（初级/中级/高级）',
        required: false
      }
    ],
    template: `
作为一位耐心的学习辅导老师，我将帮助你理解以下主题：

**学习主题**: {{topic}}
**学习水平**: {{level || "中级"}}

我会通过以下方式进行教学：

1. **概念解释**
   - 用简单易懂的语言解释核心概念
   - 提供真实世界的类比和例子
   - 突出重点和关键信息

2. **循序渐进**
   - 从基础概念开始
   - 逐步深入到更复杂的内容
   - 确保每个步骤都能理解

3. **实践应用**
   - 提供具体的例子和练习
   - 展示如何在实际中应用这些知识
   - 解释常见的应用场景

4. **学习检验**
   - 提出思考问题
   - 建议练习项目
   - 推荐进一步学习资源

请告诉我你对这个主题有什么具体的疑问，我将为你提供详细的解答和指导。
    `.trim()
  },

  'problem-solver': {
    name: 'problem-solver',
    description: '问题解决专家，提供系统性的问题分析和解决方案',
    arguments: [
      {
        name: 'problem',
        description: '要解决的问题描述',
        required: true
      },
      {
        name: 'context',
        description: '问题的背景信息',
        required: false
      }
    ],
    template: `
作为问题解决专家，我将系统性地分析和解决以下问题：

**问题描述**: {{problem}}
**背景信息**: {{context || "无额外背景信息"}}

## 问题分析框架

### 1. 问题定义
- 准确理解问题的核心
- 识别问题的范围和边界
- 区分症状和根本原因

### 2. 信息收集
- 收集相关的事实和数据
- 识别已知和未知因素
- 确定关键利益相关者

### 3. 根因分析
- 使用系统性方法找出根本原因
- 考虑多个可能的影响因素
- 验证假设和推论

### 4. 解决方案设计
- 生成多个可能的解决方案
- 评估每个方案的优缺点
- 考虑实施的可行性和风险

### 5. 实施计划
- 制定详细的行动步骤
- 确定所需资源和时间线
- 建立监控和评估机制

让我开始分析你的问题...
    `.trim()
  }
};

/**
 * 设置提示词
 */
export function setupAssistantPrompt(server: Server) {
  // 注册提示词列表处理器
  server.setRequestHandler(ListPromptsRequestSchema, async () => {
    return {
      prompts: Object.values(promptTemplates).map(template => ({
        name: template.name,
        description: template.description,
        arguments: template.arguments
      }))
    };
  });

  // 注册提示词获取处理器
  server.setRequestHandler(GetPromptRequestSchema, async (request) => {
    const { name, arguments: args } = request.params;

    const template = promptTemplates[name];
    if (!template) {
      throw new Error(`未知提示词: ${name}`);
    }

    // 渲染模板
    let renderedTemplate = template.template;
    
    // 简单的模板变量替换
    if (args) {
      Object.entries(args).forEach(([key, value]) => {
        const regex = new RegExp(`\\{\\{${key}\\}\\}`, 'g');
        renderedTemplate = renderedTemplate.replace(regex, String(value));
      });
    }

    // 处理默认值（如 {{language || "未指定"}}）
    renderedTemplate = renderedTemplate.replace(
      /\{\{(\w+)\s*\|\|\s*"([^"]*)"\}\}/g,
      (match, varName, defaultValue) => {
        return args && args[varName] ? String(args[varName]) : defaultValue;
      }
    );

    return {
      description: template.description,
      messages: [
        {
          role: 'user',
          content: {
            type: 'text',
            text: renderedTemplate
          }
        }
      ]
    };
  });
}
```

**提示词实现要点：**

1. **模板系统**：支持变量替换和默认值
2. **角色定义**：明确 AI 应该扮演的角色
3. **结构化指导**：提供清晰的思维框架
4. **灵活参数**：支持可选和必需参数

## 服务器整合和优化

现在我们需要修改主文件来正确整合所有功能：

更新 `src/index.ts`：

```typescript
#!/usr/bin/env node

import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
  ListResourcesRequestSchema,
  ReadResourceRequestSchema,
  ListPromptsRequestSchema,
  GetPromptRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';

/**
 * 统一的 MCP 服务器实现
 * 整合了工具、资源和提示词功能
 */

async function createServer() {
  const server = new Server(
    {
      name: 'my-mcp-server',
      version: '1.0.0',
    },
    {
      capabilities: {
        tools: {},
        resources: {},
        prompts: {},
      },
    }
  );

  // 工具定义
  const tools = [
    {
      name: 'calculator',
      description: '执行基本的数学计算（加、减、乘、除）',
      inputSchema: {
        type: 'object',
        properties: {
          operation: {
            type: 'string',
            enum: ['add', 'subtract', 'multiply', 'divide'],
            description: '要执行的数学运算'
          },
          a: { type: 'number', description: '第一个数字' },
          b: { type: 'number', description: '第二个数字' }
        },
        required: ['operation', 'a', 'b']
      }
    },
    {
      name: 'weather',
      description: '查询指定地点的当前天气信息',
      inputSchema: {
        type: 'object',
        properties: {
          location: {
            type: 'string',
            description: '要查询天气的城市或地点名称'
          }
        },
        required: ['location']
      }
    }
  ];

  // 注册统一的工具列表处理器
  server.setRequestHandler(ListToolsRequestSchema, async () => ({
    tools
  }));

  // 注册工具调用处理器
  server.setRequestHandler(CallToolRequestSchema, async (request) => {
    const { name, arguments: args } = request.params;

    switch (name) {
      case 'calculator':
        return handleCalculator(args as any);
      case 'weather':
        return handleWeather(args as any);
      default:
        throw new Error(`未知工具: ${name}`);
    }
  });

  // 其他处理器...（资源和提示词）
  // 这里省略了具体实现，你可以从前面的代码中复制

  return server;
}

// 工具处理函数
function handleCalculator(args: { operation: string; a: number; b: number }) {
  // 实现计算逻辑
}

function handleWeather(args: { location: string }) {
  // 实现天气查询逻辑
}

// 启动服务器的代码保持不变...
```

## 测试你的 MCP 服务器

### 本地测试

```bash
# 编译 TypeScript
npm run build

# 启动服务器
npm start
```

### 在 Claude Desktop 中测试

更新 Claude Desktop 配置文件：

```json
{
  "mcpServers": {
    "my-mcp-server": {
      "command": "node",
      "args": ["/path/to/your/my-mcp-server/dist/index.js"]
    }
  }
}
```

### 测试用例

在 Claude Desktop 中尝试以下测试：

1. **测试计算器**：
   ```
   请帮我计算 15 乘以 23 等于多少？
   ```

2. **测试天气查询**：
   ```
   请查询北京的天气情况。
   ```

3. **测试配置资源**：
   ```
   请查看服务器的配置信息。
   ```

4. **测试提示词**：
   ```
   请使用代码审查模板来分析以下 JavaScript 代码：
   function add(a, b) { return a + b; }
   ```

## 下一步

恭喜！你已经创建了一个功能完整的 MCP 服务器。接下来你可以：

1. 添加更多的工具和功能
2. 集成真实的外部 API
3. 添加数据持久化
4. 实现更复杂的业务逻辑
5. 发布到 npm 供他人使用

在下一章中，我们将通过实际案例来学习如何构建更复杂、更实用的 MCP 服务器。

---

## 完整代码仓库

你可以在 GitHub 上找到这个教程的完整代码：
[my-mcp-server-example](https://github.com/example/my-mcp-server)