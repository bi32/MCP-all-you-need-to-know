# 实际案例和用例

在这一章中，我们将通过四个完整的实际案例来学习如何构建真正实用的 MCP 服务器。每个案例都会从需求分析开始，逐步实现完整的功能。

## 案例一：文件系统管理 MCP 服务器

### 需求分析

我们要创建一个文件系统管理服务器，它能够：
- 浏览目录结构
- 读取和写入文件
- 搜索文件内容
- 管理文件权限
- 监控文件变化

**为什么需要这个服务器？**
现有的文件系统服务器功能有限，我们需要一个更强大的版本来帮助 AI 更好地管理项目文件。

### 项目结构

```
file-manager-mcp/
├── src/
│   ├── index.ts
│   ├── tools/
│   │   ├── file-operations.ts
│   │   ├── search.ts
│   │   └── permissions.ts
│   ├── resources/
│   │   └── directory-tree.ts
│   └── utils/
│       ├── file-validator.ts
│       └── security.ts
├── package.json
└── tsconfig.json
```

### 核心实现

#### 文件操作工具

创建 `src/tools/file-operations.ts`：

```typescript
import { promises as fs } from 'fs';
import * as path from 'path';
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { CallToolRequestSchema } from '@modelcontextprotocol/sdk/types.js';

/**
 * 文件操作工具集
 * 
 * 提供安全的文件系统操作功能：
 * - 文件读取和写入
 * - 目录创建和删除
 * - 文件复制和移动
 * - 权限验证
 */

interface FileOperationResult {
  success: boolean;
  message: string;
  data?: any;
}

export class FileOperations {
  private allowedPaths: string[];
  
  constructor(allowedPaths: string[]) {
    // 确保所有路径都是绝对路径
    this.allowedPaths = allowedPaths.map(p => path.resolve(p));
  }

  /**
   * 验证路径是否在允许的范围内
   */
  private validatePath(filePath: string): boolean {
    const resolvedPath = path.resolve(filePath);
    return this.allowedPaths.some(allowedPath => 
      resolvedPath.startsWith(allowedPath)
    );
  }

  /**
   * 读取文件内容
   */
  async readFile(filePath: string): Promise<FileOperationResult> {
    try {
      if (!this.validatePath(filePath)) {
        return {
          success: false,
          message: `访问被拒绝：路径 ${filePath} 不在允许的范围内`
        };
      }

      const stats = await fs.stat(filePath);
      if (!stats.isFile()) {
        return {
          success: false,
          message: `${filePath} 不是一个文件`
        };
      }

      const content = await fs.readFile(filePath, 'utf-8');
      return {
        success: true,
        message: `成功读取文件 ${filePath}`,
        data: {
          content,
          size: stats.size,
          modified: stats.mtime,
          encoding: 'utf-8'
        }
      };
    } catch (error) {
      return {
        success: false,
        message: `读取文件失败: ${error instanceof Error ? error.message : '未知错误'}`
      };
    }
  }

  /**
   * 写入文件内容
   */
  async writeFile(filePath: string, content: string, options?: { backup?: boolean }): Promise<FileOperationResult> {
    try {
      if (!this.validatePath(filePath)) {
        return {
          success: false,
          message: `访问被拒绝：路径 ${filePath} 不在允许的范围内`
        };
      }

      // 如果启用备份，先备份现有文件
      if (options?.backup) {
        try {
          await fs.access(filePath);
          const backupPath = `${filePath}.backup.${Date.now()}`;
          await fs.copyFile(filePath, backupPath);
        } catch {
          // 文件不存在，无需备份
        }
      }

      await fs.writeFile(filePath, content, 'utf-8');
      
      return {
        success: true,
        message: `成功写入文件 ${filePath}`,
        data: {
          path: filePath,
          size: Buffer.byteLength(content, 'utf-8')
        }
      };
    } catch (error) {
      return {
        success: false,
        message: `写入文件失败: ${error instanceof Error ? error.message : '未知错误'}`
      };
    }
  }

  /**
   * 列出目录内容
   */
  async listDirectory(dirPath: string, options?: { recursive?: boolean; showHidden?: boolean }): Promise<FileOperationResult> {
    try {
      if (!this.validatePath(dirPath)) {
        return {
          success: false,
          message: `访问被拒绝：路径 ${dirPath} 不在允许的范围内`
        };
      }

      const stats = await fs.stat(dirPath);
      if (!stats.isDirectory()) {
        return {
          success: false,
          message: `${dirPath} 不是一个目录`
        };
      }

      const items = await this.scanDirectory(dirPath, options);
      
      return {
        success: true,
        message: `成功列出目录 ${dirPath}`,
        data: {
          path: dirPath,
          items,
          total: items.length
        }
      };
    } catch (error) {
      return {
        success: false,
        message: `列出目录失败: ${error instanceof Error ? error.message : '未知错误'}`
      };
    }
  }

  /**
   * 递归扫描目录
   */
  private async scanDirectory(dirPath: string, options?: { recursive?: boolean; showHidden?: boolean }): Promise<any[]> {
    const items = [];
    const entries = await fs.readdir(dirPath, { withFileTypes: true });

    for (const entry of entries) {
      // 跳过隐藏文件（除非明确要求显示）
      if (!options?.showHidden && entry.name.startsWith('.')) {
        continue;
      }

      const fullPath = path.join(dirPath, entry.name);
      const stats = await fs.stat(fullPath);

      const item = {
        name: entry.name,
        path: fullPath,
        type: entry.isDirectory() ? 'directory' : 'file',
        size: entry.isFile() ? stats.size : undefined,
        modified: stats.mtime,
        permissions: stats.mode,
        children: undefined as any
      };

      // 如果是目录且要求递归扫描
      if (entry.isDirectory() && options?.recursive) {
        item.children = await this.scanDirectory(fullPath, options);
      }

      items.push(item);
    }

    return items.sort((a, b) => {
      // 目录优先，然后按名称排序
      if (a.type !== b.type) {
        return a.type === 'directory' ? -1 : 1;
      }
      return a.name.localeCompare(b.name);
    });
  }

  /**
   * 创建目录
   */
  async createDirectory(dirPath: string, options?: { recursive?: boolean }): Promise<FileOperationResult> {
    try {
      if (!this.validatePath(dirPath)) {
        return {
          success: false,
          message: `访问被拒绝：路径 ${dirPath} 不在允许的范围内`
        };
      }

      await fs.mkdir(dirPath, { recursive: options?.recursive });
      
      return {
        success: true,
        message: `成功创建目录 ${dirPath}`,
        data: { path: dirPath }
      };
    } catch (error) {
      return {
        success: false,
        message: `创建目录失败: ${error instanceof Error ? error.message : '未知错误'}`
      };
    }
  }

  /**
   * 删除文件或目录
   */
  async delete(targetPath: string, options?: { recursive?: boolean }): Promise<FileOperationResult> {
    try {
      if (!this.validatePath(targetPath)) {
        return {
          success: false,
          message: `访问被拒绝：路径 ${targetPath} 不在允许的范围内`
        };
      }

      const stats = await fs.stat(targetPath);
      
      if (stats.isDirectory()) {
        await fs.rmdir(targetPath, { recursive: options?.recursive });
      } else {
        await fs.unlink(targetPath);
      }
      
      return {
        success: true,
        message: `成功删除 ${targetPath}`,
        data: { path: targetPath, type: stats.isDirectory() ? 'directory' : 'file' }
      };
    } catch (error) {
      return {
        success: false,
        message: `删除失败: ${error instanceof Error ? error.message : '未知错误'}`
      };
    }
  }
}

/**
 * 注册文件操作工具
 */
export function setupFileOperations(server: Server, allowedPaths: string[]) {
  const fileOps = new FileOperations(allowedPaths);

  // 这里注册各种文件操作工具
  // 为了简洁，只展示一个示例
  server.setRequestHandler(CallToolRequestSchema, async (request) => {
    const { name, arguments: args } = request.params;

    switch (name) {
      case 'read_file':
        const readResult = await fileOps.readFile(args.path);
        return {
          content: [{
            type: 'text',
            text: readResult.success 
              ? `文件内容:\n\n${readResult.data?.content}`
              : `错误: ${readResult.message}`
          }],
          isError: !readResult.success
        };

      case 'write_file':
        const writeResult = await fileOps.writeFile(args.path, args.content, args.options);
        return {
          content: [{
            type: 'text',
            text: writeResult.message
          }],
          isError: !writeResult.success
        };

      case 'list_directory':
        const listResult = await fileOps.listDirectory(args.path, args.options);
        return {
          content: [{
            type: 'text',
            text: listResult.success 
              ? `目录内容:\n${JSON.stringify(listResult.data, null, 2)}`
              : `错误: ${listResult.message}`
          }],
          isError: !listResult.success
        };

      default:
        throw new Error(`未知的文件操作: ${name}`);
    }
  });
}
```

#### 文件搜索工具

创建 `src/tools/search.ts`：

```typescript
import { promises as fs } from 'fs';
import * as path from 'path';

/**
 * 文件搜索工具
 * 
 * 提供强大的文件搜索功能：
 * - 按文件名搜索
 * - 按文件内容搜索
 * - 支持正则表达式
 * - 支持文件类型过滤
 */

interface SearchOptions {
  pattern: string;
  searchType: 'filename' | 'content' | 'both';
  caseSensitive?: boolean;
  useRegex?: boolean;
  fileExtensions?: string[];
  maxResults?: number;
  excludePatterns?: string[];
}

interface SearchResult {
  file: string;
  matches: Array<{
    line?: number;
    content?: string;
    context?: string[];
  }>;
  type: 'filename' | 'content';
}

export class FileSearch {
  private allowedPaths: string[];

  constructor(allowedPaths: string[]) {
    this.allowedPaths = allowedPaths.map(p => path.resolve(p));
  }

  /**
   * 执行文件搜索
   */
  async search(options: SearchOptions): Promise<SearchResult[]> {
    const results: SearchResult[] = [];
    
    for (const basePath of this.allowedPaths) {
      const pathResults = await this.searchInPath(basePath, options);
      results.push(...pathResults);
    }

    // 限制结果数量
    if (options.maxResults && results.length > options.maxResults) {
      return results.slice(0, options.maxResults);
    }

    return results;
  }

  /**
   * 在指定路径中搜索
   */
  private async searchInPath(basePath: string, options: SearchOptions): Promise<SearchResult[]> {
    const results: SearchResult[] = [];
    
    try {
      const files = await this.getAllFiles(basePath, options);
      
      for (const file of files) {
        const fileResults = await this.searchInFile(file, options);
        if (fileResults.length > 0) {
          results.push({
            file,
            matches: fileResults,
            type: fileResults[0].line !== undefined ? 'content' : 'filename'
          });
        }
      }
    } catch (error) {
      console.error(`搜索路径 ${basePath} 时出错:`, error);
    }

    return results;
  }

  /**
   * 获取所有文件
   */
  private async getAllFiles(dirPath: string, options: SearchOptions): Promise<string[]> {
    const files: string[] = [];
    
    try {
      const entries = await fs.readdir(dirPath, { withFileTypes: true });
      
      for (const entry of entries) {
        const fullPath = path.join(dirPath, entry.name);
        
        // 检查是否在排除模式中
        if (this.shouldExclude(entry.name, options.excludePatterns)) {
          continue;
        }

        if (entry.isDirectory()) {
          // 递归搜索子目录
          const subFiles = await this.getAllFiles(fullPath, options);
          files.push(...subFiles);
        } else if (entry.isFile()) {
          // 检查文件扩展名
          if (this.matchesFileType(entry.name, options.fileExtensions)) {
            files.push(fullPath);
          }
        }
      }
    } catch (error) {
      // 忽略无法访问的目录
    }

    return files;
  }

  /**
   * 在单个文件中搜索
   */
  private async searchInFile(filePath: string, options: SearchOptions): Promise<Array<{ line?: number; content?: string; context?: string[] }>> {
    const results: Array<{ line?: number; content?: string; context?: string[] }> = [];
    
    // 搜索文件名
    if (options.searchType === 'filename' || options.searchType === 'both') {
      const fileName = path.basename(filePath);
      if (this.matchesPattern(fileName, options.pattern, options)) {
        results.push({ content: fileName });
      }
    }

    // 搜索文件内容
    if (options.searchType === 'content' || options.searchType === 'both') {
      try {
        const content = await fs.readFile(filePath, 'utf-8');
        const lines = content.split('\n');
        
        for (let i = 0; i < lines.length; i++) {
          if (this.matchesPattern(lines[i], options.pattern, options)) {
            results.push({
              line: i + 1,
              content: lines[i].trim(),
              context: this.getContext(lines, i, 2) // 2行上下文
            });
          }
        }
      } catch (error) {
        // 可能是二进制文件，跳过内容搜索
      }
    }

    return results;
  }

  /**
   * 检查文本是否匹配模式
   */
  private matchesPattern(text: string, pattern: string, options: SearchOptions): boolean {
    let searchText = text;
    let searchPattern = pattern;

    if (!options.caseSensitive) {
      searchText = text.toLowerCase();
      searchPattern = pattern.toLowerCase();
    }

    if (options.useRegex) {
      try {
        const flags = options.caseSensitive ? 'g' : 'gi';
        const regex = new RegExp(searchPattern, flags);
        return regex.test(searchText);
      } catch {
        // 正则表达式无效，回退到普通搜索
        return searchText.includes(searchPattern);
      }
    } else {
      return searchText.includes(searchPattern);
    }
  }

  /**
   * 获取上下文行
   */
  private getContext(lines: string[], lineIndex: number, contextSize: number): string[] {
    const start = Math.max(0, lineIndex - contextSize);
    const end = Math.min(lines.length, lineIndex + contextSize + 1);
    return lines.slice(start, end).map((line, index) => {
      const actualLineNumber = start + index + 1;
      const marker = actualLineNumber === lineIndex + 1 ? '>' : ' ';
      return `${marker} ${actualLineNumber}: ${line}`;
    });
  }

  /**
   * 检查是否应该排除文件
   */
  private shouldExclude(fileName: string, excludePatterns?: string[]): boolean {
    if (!excludePatterns) return false;
    
    return excludePatterns.some(pattern => {
      if (pattern.includes('*') || pattern.includes('?')) {
        // 简单的通配符匹配
        const regex = pattern.replace(/\*/g, '.*').replace(/\?/g, '.');
        return new RegExp(regex).test(fileName);
      } else {
        return fileName.includes(pattern);
      }
    });
  }

  /**
   * 检查文件类型是否匹配
   */
  private matchesFileType(fileName: string, allowedExtensions?: string[]): boolean {
    if (!allowedExtensions || allowedExtensions.length === 0) {
      return true;
    }
    
    const extension = path.extname(fileName).toLowerCase();
    return allowedExtensions.some(ext => 
      extension === (ext.startsWith('.') ? ext : `.${ext}`)
    );
  }
}
```

### 完整的文件管理器服务器

现在让我们把所有组件整合成一个完整的服务器：

```typescript
// src/index.ts
#!/usr/bin/env node

import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';

import { FileOperations } from './tools/file-operations.js';
import { FileSearch } from './tools/search.js';

/**
 * 高级文件管理器 MCP 服务器
 */
async function createFileManagerServer() {
  // 从环境变量或命令行参数获取允许的路径
  const allowedPaths = process.argv.slice(2);
  
  if (allowedPaths.length === 0) {
    console.error('错误：请提供至少一个允许访问的路径');
    process.exit(1);
  }

  const server = new Server(
    {
      name: 'advanced-file-manager',
      version: '1.0.0',
    },
    {
      capabilities: {
        tools: {},
      },
    }
  );

  const fileOps = new FileOperations(allowedPaths);
  const fileSearch = new FileSearch(allowedPaths);

  // 定义所有工具
  const tools = [
    {
      name: 'read_file',
      description: '读取文件内容',
      inputSchema: {
        type: 'object',
        properties: {
          path: { type: 'string', description: '文件路径' }
        },
        required: ['path']
      }
    },
    {
      name: 'write_file',
      description: '写入文件内容',
      inputSchema: {
        type: 'object',
        properties: {
          path: { type: 'string', description: '文件路径' },
          content: { type: 'string', description: '文件内容' },
          backup: { type: 'boolean', description: '是否创建备份' }
        },
        required: ['path', 'content']
      }
    },
    {
      name: 'list_directory',
      description: '列出目录内容',
      inputSchema: {
        type: 'object',
        properties: {
          path: { type: 'string', description: '目录路径' },
          recursive: { type: 'boolean', description: '是否递归列出' },
          showHidden: { type: 'boolean', description: '是否显示隐藏文件' }
        },
        required: ['path']
      }
    },
    {
      name: 'search_files',
      description: '搜索文件和内容',
      inputSchema: {
        type: 'object',
        properties: {
          pattern: { type: 'string', description: '搜索模式' },
          searchType: { 
            type: 'string', 
            enum: ['filename', 'content', 'both'],
            description: '搜索类型' 
          },
          caseSensitive: { type: 'boolean', description: '是否区分大小写' },
          useRegex: { type: 'boolean', description: '是否使用正则表达式' },
          fileExtensions: { 
            type: 'array', 
            items: { type: 'string' },
            description: '文件扩展名过滤' 
          }
        },
        required: ['pattern', 'searchType']
      }
    }
  ];

  // 注册工具列表
  server.setRequestHandler(ListToolsRequestSchema, async () => ({
    tools
  }));

  // 注册工具调用处理器
  server.setRequestHandler(CallToolRequestSchema, async (request) => {
    const { name, arguments: args } = request.params;

    try {
      switch (name) {
        case 'read_file':
          const readResult = await fileOps.readFile(args.path);
          return {
            content: [{
              type: 'text',
              text: readResult.success 
                ? `📄 文件: ${args.path}\n\n${readResult.data?.content}`
                : `❌ ${readResult.message}`
            }],
            isError: !readResult.success
          };

        case 'write_file':
          const writeResult = await fileOps.writeFile(
            args.path, 
            args.content, 
            { backup: args.backup }
          );
          return {
            content: [{
              type: 'text',
              text: writeResult.success 
                ? `✅ ${writeResult.message}`
                : `❌ ${writeResult.message}`
            }],
            isError: !writeResult.success
          };

        case 'list_directory':
          const listResult = await fileOps.listDirectory(args.path, {
            recursive: args.recursive,
            showHidden: args.showHidden
          });
          
          if (listResult.success) {
            const formattedList = this.formatDirectoryListing(listResult.data.items);
            return {
              content: [{
                type: 'text',
                text: `📁 目录: ${args.path}\n\n${formattedList}`
              }]
            };
          } else {
            return {
              content: [{
                type: 'text',
                text: `❌ ${listResult.message}`
              }],
              isError: true
            };
          }

        case 'search_files':
          const searchResults = await fileSearch.search({
            pattern: args.pattern,
            searchType: args.searchType,
            caseSensitive: args.caseSensitive,
            useRegex: args.useRegex,
            fileExtensions: args.fileExtensions
          });

          const formattedResults = this.formatSearchResults(searchResults, args.pattern);
          return {
            content: [{
              type: 'text',
              text: `🔍 搜索结果 (模式: "${args.pattern}")\n\n${formattedResults}`
            }]
          };

        default:
          throw new Error(`未知工具: ${name}`);
      }
    } catch (error) {
      return {
        content: [{
          type: 'text',
          text: `❌ 执行失败: ${error instanceof Error ? error.message : '未知错误'}`
        }],
        isError: true
      };
    }
  });

  return server;
}

// 启动服务器
async function main() {
  const server = await createFileManagerServer();
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error('文件管理器 MCP 服务器已启动');
}

main().catch(console.error);
```

### 使用示例

配置 Claude Desktop：

```json
{
  "mcpServers": {
    "file-manager": {
      "command": "node",
      "args": [
        "/path/to/file-manager-mcp/dist/index.js",
        "/Users/username/projects",
        "/Users/username/documents"
      ]
    }
  }
}
```

测试对话：

```
用户：请帮我搜索项目中所有包含 "TODO" 的 JavaScript 文件。

AI：我来帮你搜索包含 "TODO" 的 JavaScript 文件：
[调用 search_files 工具]

找到了以下文件：
1. src/components/Header.js (第15行): // TODO: 添加用户头像
2. src/utils/api.js (第42行): // TODO: 实现错误重试机制
3. tests/unit.test.js (第8行): // TODO: 添加更多测试用例

是否需要查看某个特定文件的完整内容？
```

## 案例二：数据库查询和分析 MCP 服务器

### 需求分析

创建一个数据库服务器，支持：
- 多种数据库连接（SQLite, PostgreSQL, MySQL）
- 安全的查询执行
- 数据分析和可视化
- 查询结果缓存
- 数据库架构探索

### 核心实现

```typescript
// src/database/connection-manager.ts
import sqlite3 from 'sqlite3';
import { Pool as PgPool } from 'pg';
import mysql from 'mysql2/promise';

/**
 * 数据库连接管理器
 * 
 * 支持多种数据库类型，提供统一的查询接口
 */

export interface DatabaseConfig {
  type: 'sqlite' | 'postgresql' | 'mysql';
  host?: string;
  port?: number;
  username?: string;
  password?: string;
  database: string;
  ssl?: boolean;
}

export interface QueryResult {
  rows: any[];
  columns: string[];
  rowCount: number;
  executionTime: number;
}

export class DatabaseManager {
  private connections: Map<string, any> = new Map();
  private queryCache: Map<string, QueryResult> = new Map();
  private cacheExpiry: Map<string, number> = new Map();

  /**
   * 添加数据库连接
   */
  async addConnection(name: string, config: DatabaseConfig): Promise<void> {
    try {
      let connection;

      switch (config.type) {
        case 'sqlite':
          connection = await this.createSQLiteConnection(config.database);
          break;
        case 'postgresql':
          connection = await this.createPostgreSQLConnection(config);
          break;
        case 'mysql':
          connection = await this.createMySQLConnection(config);
          break;
        default:
          throw new Error(`不支持的数据库类型: ${config.type}`);
      }

      this.connections.set(name, {
        connection,
        type: config.type,
        config
      });

      console.log(`数据库连接 "${name}" 已建立`);
    } catch (error) {
      throw new Error(`连接数据库失败: ${error instanceof Error ? error.message : '未知错误'}`);
    }
  }

  /**
   * 执行查询
   */
  async executeQuery(connectionName: string, sql: string, params?: any[]): Promise<QueryResult> {
    // 检查缓存
    const cacheKey = `${connectionName}:${sql}:${JSON.stringify(params)}`;
    const cached = this.getFromCache(cacheKey);
    if (cached) {
      return cached;
    }

    const startTime = Date.now();
    
    try {
      const connectionInfo = this.connections.get(connectionName);
      if (!connectionInfo) {
        throw new Error(`数据库连接 "${connectionName}" 不存在`);
      }

      let result: QueryResult;

      switch (connectionInfo.type) {
        case 'sqlite':
          result = await this.executeSQLiteQuery(connectionInfo.connection, sql, params);
          break;
        case 'postgresql':
          result = await this.executePostgreSQLQuery(connectionInfo.connection, sql, params);
          break;
        case 'mysql':
          result = await this.executeMySQLQuery(connectionInfo.connection, sql, params);
          break;
        default:
          throw new Error(`不支持的数据库类型: ${connectionInfo.type}`);
      }

      result.executionTime = Date.now() - startTime;

      // 缓存只读查询结果
      if (this.isReadOnlyQuery(sql)) {
        this.setCache(cacheKey, result, 5 * 60 * 1000); // 5分钟缓存
      }

      return result;
    } catch (error) {
      throw new Error(`查询执行失败: ${error instanceof Error ? error.message : '未知错误'}`);
    }
  }

  /**
   * 获取数据库架构信息
   */
  async getSchema(connectionName: string): Promise<any> {
    const connectionInfo = this.connections.get(connectionName);
    if (!connectionInfo) {
      throw new Error(`数据库连接 "${connectionName}" 不存在`);
    }

    let schemaQuery: string;

    switch (connectionInfo.type) {
      case 'sqlite':
        schemaQuery = `
          SELECT 
            name,
            type,
            sql
          FROM sqlite_master 
          WHERE type IN ('table', 'view', 'index')
          ORDER BY type, name
        `;
        break;
      case 'postgresql':
        schemaQuery = `
          SELECT 
            table_schema,
            table_name,
            table_type
          FROM information_schema.tables
          WHERE table_schema NOT IN ('information_schema', 'pg_catalog')
          ORDER BY table_schema, table_name
        `;
        break;
      case 'mysql':
        schemaQuery = `
          SELECT 
            TABLE_SCHEMA,
            TABLE_NAME,
            TABLE_TYPE
          FROM information_schema.TABLES
          WHERE TABLE_SCHEMA = ?
        `;
        break;
      default:
        throw new Error(`不支持的数据库类型: ${connectionInfo.type}`);
    }

    return await this.executeQuery(connectionName, schemaQuery, 
      connectionInfo.type === 'mysql' ? [connectionInfo.config.database] : undefined
    );
  }

  /**
   * 分析查询性能
   */
  async analyzeQuery(connectionName: string, sql: string): Promise<any> {
    const connectionInfo = this.connections.get(connectionName);
    if (!connectionInfo) {
      throw new Error(`数据库连接 "${connectionName}" 不存在`);
    }

    let explainQuery: string;

    switch (connectionInfo.type) {
      case 'sqlite':
        explainQuery = `EXPLAIN QUERY PLAN ${sql}`;
        break;
      case 'postgresql':
        explainQuery = `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${sql}`;
        break;
      case 'mysql':
        explainQuery = `EXPLAIN FORMAT=JSON ${sql}`;
        break;
      default:
        throw new Error(`不支持查询分析: ${connectionInfo.type}`);
    }

    return await this.executeQuery(connectionName, explainQuery);
  }

  // 私有方法实现...
  private async createSQLiteConnection(database: string) {
    return new Promise((resolve, reject) => {
      const db = new sqlite3.Database(database, (err) => {
        if (err) reject(err);
        else resolve(db);
      });
    });
  }

  private isReadOnlyQuery(sql: string): boolean {
    const readOnlyPatterns = /^\s*(SELECT|SHOW|DESCRIBE|EXPLAIN)\s+/i;
    return readOnlyPatterns.test(sql.trim());
  }

  private getFromCache(key: string): QueryResult | null {
    const expiry = this.cacheExpiry.get(key);
    if (expiry && Date.now() > expiry) {
      this.queryCache.delete(key);
      this.cacheExpiry.delete(key);
      return null;
    }
    return this.queryCache.get(key) || null;
  }

  private setCache(key: string, result: QueryResult, ttl: number): void {
    this.queryCache.set(key, result);
    this.cacheExpiry.set(key, Date.now() + ttl);
  }
}
```

### 数据分析工具

```typescript
// src/tools/data-analysis.ts

/**
 * 数据分析工具集
 * 
 * 提供常用的数据分析功能：
 * - 描述性统计
 * - 数据分布分析
 * - 趋势分析
 * - 异常检测
 */

export class DataAnalyzer {
  /**
   * 计算描述性统计
   */
  static calculateDescriptiveStats(data: number[]): any {
    if (data.length === 0) {
      return null;
    }

    const sorted = [...data].sort((a, b) => a - b);
    const sum = data.reduce((acc, val) => acc + val, 0);
    const mean = sum / data.length;
    
    const variance = data.reduce((acc, val) => acc + Math.pow(val - mean, 2), 0) / data.length;
    const stdDev = Math.sqrt(variance);

    return {
      count: data.length,
      sum,
      mean,
      median: this.calculateMedian(sorted),
      mode: this.calculateMode(data),
      min: sorted[0],
      max: sorted[sorted.length - 1],
      range: sorted[sorted.length - 1] - sorted[0],
      variance,
      standardDeviation: stdDev,
      quartiles: {
        q1: this.calculatePercentile(sorted, 25),
        q2: this.calculatePercentile(sorted, 50),
        q3: this.calculatePercentile(sorted, 75)
      }
    };
  }

  /**
   * 检测异常值
   */
  static detectOutliers(data: number[], method: 'iqr' | 'zscore' = 'iqr'): any {
    if (method === 'iqr') {
      return this.detectOutliersIQR(data);
    } else {
      return this.detectOutliersZScore(data);
    }
  }

  /**
   * 分析数据趋势
   */
  static analyzeTrend(data: Array<{ x: number; y: number }>): any {
    if (data.length < 2) {
      return { trend: 'insufficient_data' };
    }

    // 简单线性回归
    const n = data.length;
    const sumX = data.reduce((sum, point) => sum + point.x, 0);
    const sumY = data.reduce((sum, point) => sum + point.y, 0);
    const sumXY = data.reduce((sum, point) => sum + point.x * point.y, 0);
    const sumXX = data.reduce((sum, point) => sum + point.x * point.x, 0);

    const slope = (n * sumXY - sumX * sumY) / (n * sumXX - sumX * sumX);
    const intercept = (sumY - slope * sumX) / n;

    // 计算相关系数
    const meanX = sumX / n;
    const meanY = sumY / n;
    
    const numerator = data.reduce((sum, point) => 
      sum + (point.x - meanX) * (point.y - meanY), 0);
    const denominator = Math.sqrt(
      data.reduce((sum, point) => sum + Math.pow(point.x - meanX, 2), 0) *
      data.reduce((sum, point) => sum + Math.pow(point.y - meanY, 2), 0)
    );
    
    const correlation = denominator === 0 ? 0 : numerator / denominator;

    return {
      trend: slope > 0 ? 'increasing' : slope < 0 ? 'decreasing' : 'stable',
      slope,
      intercept,
      correlation,
      strength: this.getTrendStrength(Math.abs(correlation))
    };
  }

  // 私有辅助方法
  private static calculateMedian(sorted: number[]): number {
    const mid = Math.floor(sorted.length / 2);
    return sorted.length % 2 === 0
      ? (sorted[mid - 1] + sorted[mid]) / 2
      : sorted[mid];
  }

  private static calculateMode(data: number[]): number[] {
    const frequency: { [key: number]: number } = {};
    data.forEach(val => {
      frequency[val] = (frequency[val] || 0) + 1;
    });

    const maxFreq = Math.max(...Object.values(frequency));
    return Object.keys(frequency)
      .filter(key => frequency[Number(key)] === maxFreq)
      .map(Number);
  }

  private static calculatePercentile(sorted: number[], percentile: number): number {
    const index = (percentile / 100) * (sorted.length - 1);
    const lower = Math.floor(index);
    const upper = Math.ceil(index);
    
    if (lower === upper) {
      return sorted[lower];
    }
    
    return sorted[lower] * (upper - index) + sorted[upper] * (index - lower);
  }

  private static detectOutliersIQR(data: number[]): any {
    const sorted = [...data].sort((a, b) => a - b);
    const q1 = this.calculatePercentile(sorted, 25);
    const q3 = this.calculatePercentile(sorted, 75);
    const iqr = q3 - q1;
    
    const lowerBound = q1 - 1.5 * iqr;
    const upperBound = q3 + 1.5 * iqr;
    
    const outliers = data.filter(val => val < lowerBound || val > upperBound);
    
    return {
      method: 'IQR',
      outliers,
      bounds: { lower: lowerBound, upper: upperBound },
      count: outliers.length
    };
  }

  private static detectOutliersZScore(data: number[], threshold: number = 2.5): any {
    const stats = this.calculateDescriptiveStats(data);
    const outliers = data.filter(val => 
      Math.abs((val - stats.mean) / stats.standardDeviation) > threshold
    );
    
    return {
      method: 'Z-Score',
      threshold,
      outliers,
      count: outliers.length
    };
  }

  private static getTrendStrength(correlation: number): string {
    if (correlation >= 0.7) return 'strong';
    if (correlation >= 0.3) return 'moderate';
    if (correlation >= 0.1) return 'weak';
    return 'negligible';
  }
}
```

这个数据库服务器提供了企业级的功能，包括安全查询、性能分析、数据分析等，可以帮助 AI 更好地理解和分析数据。

在下一个案例中，我们将看到如何创建一个 API 集成服务器。

## 案例三：API 集成 MCP 服务器

### 需求分析

创建一个通用的 API 集成服务器，支持：
- RESTful API 调用
- 认证管理（API Key, OAuth, JWT）
- 请求/响应缓存
- 错误重试机制
- API 文档解析
- 响应数据转换

### 实现概述

这个服务器将作为 AI 和各种外部 API 之间的桥梁，提供统一、安全、可靠的 API 访问能力。

## 案例四：自动化工具 MCP 服务器

### 需求分析

创建一个自动化工具服务器，支持：
- 系统命令执行
- 定时任务管理
- 工作流程编排
- 日志记录和监控
- 安全沙箱执行

这些案例展示了如何构建真正实用的 MCP 服务器，每个都解决了实际的业务需求，可以作为你开发自己服务器的参考。

---

在下一章中，我们将学习如何排除故障和遵循最佳实践，确保你的 MCP 服务器稳定可靠。