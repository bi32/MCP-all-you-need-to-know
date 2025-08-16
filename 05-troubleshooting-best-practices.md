# 故障排除和最佳实践

在这一章中，我们将学习如何诊断和解决 MCP 开发中的常见问题，以及如何遵循最佳实践来构建高质量、可维护的 MCP 服务器。

## 常见问题和解决方案

### 连接问题

#### 问题：MCP 服务器无法连接到客户端

**症状表现：**
- Claude Desktop 中看不到 MCP 服务器提供的工具
- 控制台显示连接超时或拒绝连接
- 服务器启动后立即退出

**诊断步骤：**

1. **检查配置文件语法**

```bash
# 验证 JSON 配置文件语法
python -c "import json; print(json.load(open('claude_desktop_config.json')))"

# 或者使用 jq 工具
jq '.' claude_desktop_config.json
```

2. **验证服务器可执行性**

```bash
# 测试服务器是否可以正常启动
node /path/to/your/mcp-server/dist/index.js --help

# 检查 npm 包是否正确安装
npm list -g @your-scope/your-mcp-server
```

3. **检查路径和权限**

```bash
# 确保文件存在且有执行权限
ls -la /path/to/your/mcp-server/dist/index.js

# 检查 Node.js 路径
which node
which npx
```

**常见解决方案：**

```json
// 正确的配置示例
{
  "mcpServers": {
    "my-server": {
      "command": "node",
      "args": [
        "/absolute/path/to/server/dist/index.js",
        "--allowed-path=/Users/username/projects"
      ],
      "env": {
        "NODE_ENV": "production"
      }
    }
  }
}
```

**最佳实践：**
- 始终使用绝对路径
- 验证所有路径确实存在
- 在配置中添加必要的环境变量
- 使用日志记录来跟踪启动过程

#### 问题：权限被拒绝

**症状表现：**
- 工具调用失败，返回权限错误
- 无法访问指定的文件或目录
- 系统级操作被阻止

**安全验证清单：**

```typescript
// 实现安全的路径验证
class SecurityValidator {
  private allowedPaths: string[];
  
  constructor(allowedPaths: string[]) {
    // 确保所有路径都是绝对路径并标准化
    this.allowedPaths = allowedPaths.map(p => path.resolve(p));
  }

  validatePath(requestedPath: string): boolean {
    try {
      const resolvedPath = path.resolve(requestedPath);
      
      // 检查路径是否在允许范围内
      const isAllowed = this.allowedPaths.some(allowedPath => 
        resolvedPath.startsWith(allowedPath)
      );
      
      if (!isAllowed) {
        console.warn(`拒绝访问路径: ${resolvedPath}`);
        return false;
      }

      // 检查文件是否存在
      fs.accessSync(resolvedPath, fs.constants.R_OK);
      return true;
    } catch (error) {
      console.error(`路径验证失败: ${error.message}`);
      return false;
    }
  }

  validateOperation(operation: string, context: any): boolean {
    // 根据操作类型进行额外的安全检查
    switch (operation) {
      case 'write':
        return this.validateWriteOperation(context);
      case 'delete':
        return this.validateDeleteOperation(context);
      case 'execute':
        return this.validateExecuteOperation(context);
      default:
        return true;
    }
  }

  private validateWriteOperation(context: any): boolean {
    // 检查写入权限和文件大小限制
    const maxFileSize = 10 * 1024 * 1024; // 10MB
    if (context.content && Buffer.byteLength(context.content) > maxFileSize) {
      console.warn(`文件大小超过限制: ${Buffer.byteLength(context.content)} bytes`);
      return false;
    }
    return true;
  }

  private validateDeleteOperation(context: any): boolean {
    // 防止删除重要系统文件
    const protectedPaths = [
      '/etc',
      '/bin',
      '/usr/bin',
      '/System'
    ];
    
    const targetPath = path.resolve(context.path);
    return !protectedPaths.some(protected => 
      targetPath.startsWith(protected)
    );
  }

  private validateExecuteOperation(context: any): boolean {
    // 限制可执行的命令
    const allowedCommands = ['git', 'npm', 'node', 'python'];
    const command = context.command.split(' ')[0];
    return allowedCommands.includes(command);
  }
}
```

### 性能问题

#### 问题：响应速度慢

**诊断工具：**

```typescript
// 性能监控中间件
class PerformanceMonitor {
  private metrics: Map<string, any[]> = new Map();

  startTiming(operationId: string): string {
    const startTime = process.hrtime.bigint();
    return `${operationId}-${startTime}`;
  }

  endTiming(timingId: string, operationName: string): void {
    const [operationId, startTime] = timingId.split('-');
    const endTime = process.hrtime.bigint();
    const duration = Number(endTime - BigInt(startTime)) / 1e6; // 转换为毫秒

    if (!this.metrics.has(operationName)) {
      this.metrics.set(operationName, []);
    }

    this.metrics.get(operationName)!.push({
      timestamp: new Date(),
      duration,
      operationId
    });

    // 如果操作时间超过阈值，记录警告
    if (duration > 1000) { // 1秒
      console.warn(`慢操作检测: ${operationName} 耗时 ${duration.toFixed(2)}ms`);
    }
  }

  getMetrics(operationName?: string): any {
    if (operationName) {
      const data = this.metrics.get(operationName) || [];
      return this.calculateStats(data);
    }

    const result: any = {};
    for (const [name, data] of this.metrics.entries()) {
      result[name] = this.calculateStats(data);
    }
    return result;
  }

  private calculateStats(data: any[]): any {
    if (data.length === 0) return null;

    const durations = data.map(d => d.duration);
    const total = durations.reduce((sum, d) => sum + d, 0);
    const avg = total / durations.length;
    
    durations.sort((a, b) => a - b);
    const median = durations[Math.floor(durations.length / 2)];
    
    return {
      count: data.length,
      avgDuration: avg.toFixed(2),
      medianDuration: median.toFixed(2),
      minDuration: durations[0].toFixed(2),
      maxDuration: durations[durations.length - 1].toFixed(2),
      lastOperation: data[data.length - 1].timestamp
    };
  }
}

// 在服务器中使用性能监控
const performanceMonitor = new PerformanceMonitor();

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const timingId = performanceMonitor.startTiming(request.id);
  
  try {
    const result = await handleToolCall(request);
    return result;
  } finally {
    performanceMonitor.endTiming(timingId, request.params.name);
  }
});
```

**性能优化策略：**

1. **实现智能缓存**

```typescript
class SmartCache {
  private cache: Map<string, any> = new Map();
  private expiry: Map<string, number> = new Map();
  private hitCount: Map<string, number> = new Map();
  private maxSize: number;

  constructor(maxSize: number = 1000) {
    this.maxSize = maxSize;
  }

  set(key: string, value: any, ttl: number = 300000): void { // 默认5分钟
    // 如果缓存已满，删除最少使用的项
    if (this.cache.size >= this.maxSize) {
      this.evictLeastUsed();
    }

    this.cache.set(key, value);
    this.expiry.set(key, Date.now() + ttl);
    this.hitCount.set(key, 0);
  }

  get(key: string): any | null {
    // 检查是否过期
    const expireTime = this.expiry.get(key);
    if (expireTime && Date.now() > expireTime) {
      this.delete(key);
      return null;
    }

    const value = this.cache.get(key);
    if (value !== undefined) {
      // 增加命中次数
      this.hitCount.set(key, (this.hitCount.get(key) || 0) + 1);
      return value;
    }

    return null;
  }

  delete(key: string): void {
    this.cache.delete(key);
    this.expiry.delete(key);
    this.hitCount.delete(key);
  }

  private evictLeastUsed(): void {
    let leastUsedKey = '';
    let minHits = Infinity;

    for (const [key, hits] of this.hitCount.entries()) {
      if (hits < minHits) {
        minHits = hits;
        leastUsedKey = key;
      }
    }

    if (leastUsedKey) {
      this.delete(leastUsedKey);
    }
  }

  getStats(): any {
    return {
      size: this.cache.size,
      maxSize: this.maxSize,
      hitRates: Array.from(this.hitCount.entries())
        .map(([key, hits]) => ({ key, hits }))
        .sort((a, b) => b.hits - a.hits)
    };
  }
}
```

2. **异步处理和批量操作**

```typescript
// 批量处理工具
class BatchProcessor {
  private queue: any[] = [];
  private processing: boolean = false;
  private batchSize: number;
  private flushInterval: number;
  private timer: NodeJS.Timeout | null = null;

  constructor(batchSize: number = 10, flushInterval: number = 1000) {
    this.batchSize = batchSize;
    this.flushInterval = flushInterval;
  }

  add(item: any): Promise<any> {
    return new Promise((resolve, reject) => {
      this.queue.push({ item, resolve, reject });
      
      if (this.queue.length >= this.batchSize) {
        this.flush();
      } else if (!this.timer) {
        this.timer = setTimeout(() => this.flush(), this.flushInterval);
      }
    });
  }

  private async flush(): void {
    if (this.processing || this.queue.length === 0) {
      return;
    }

    this.processing = true;
    
    if (this.timer) {
      clearTimeout(this.timer);
      this.timer = null;
    }

    const batch = this.queue.splice(0, this.batchSize);
    
    try {
      await this.processBatch(batch);
    } catch (error) {
      batch.forEach(({ reject }) => reject(error));
    }

    this.processing = false;

    // 如果还有待处理的项目，继续处理
    if (this.queue.length > 0) {
      setImmediate(() => this.flush());
    }
  }

  private async processBatch(batch: any[]): Promise<void> {
    // 这里实现具体的批量处理逻辑
    // 例如：批量数据库查询、批量文件操作等
    console.log(`处理批量操作，大小: ${batch.length}`);
    
    // 模拟批量处理
    for (const { item, resolve } of batch) {
      try {
        const result = await this.processItem(item);
        resolve(result);
      } catch (error) {
        batch.forEach(({ reject }) => reject(error));
        break;
      }
    }
  }

  private async processItem(item: any): Promise<any> {
    // 处理单个项目的逻辑
    return item;
  }
}
```

### 错误处理和日志记录

#### 统一错误处理

```typescript
// 错误类型定义
export enum ErrorType {
  VALIDATION_ERROR = 'VALIDATION_ERROR',
  PERMISSION_DENIED = 'PERMISSION_DENIED',
  RESOURCE_NOT_FOUND = 'RESOURCE_NOT_FOUND',
  EXTERNAL_API_ERROR = 'EXTERNAL_API_ERROR',
  INTERNAL_SERVER_ERROR = 'INTERNAL_SERVER_ERROR'
}

export class MCPError extends Error {
  public readonly type: ErrorType;
  public readonly code: string;
  public readonly details?: any;

  constructor(type: ErrorType, message: string, code?: string, details?: any) {
    super(message);
    this.type = type;
    this.code = code || type;
    this.details = details;
    this.name = 'MCPError';
  }

  toJSON(): any {
    return {
      type: this.type,
      code: this.code,
      message: this.message,
      details: this.details
    };
  }
}

// 错误处理中间件
export class ErrorHandler {
  static handle(error: any): any {
    if (error instanceof MCPError) {
      return {
        content: [{
          type: 'text',
          text: `❌ ${error.message}`
        }],
        isError: true,
        metadata: {
          errorType: error.type,
          errorCode: error.code,
          details: error.details
        }
      };
    }

    // 处理其他类型的错误
    console.error('未知错误:', error);
    return {
      content: [{
        type: 'text',
        text: '❌ 发生了未知错误，请检查服务器日志'
      }],
      isError: true
    };
  }

  static validateInput(schema: any, input: any): void {
    // 这里可以使用 JSON Schema 验证库
    // 简化版本的输入验证
    if (schema.required) {
      for (const field of schema.required) {
        if (!(field in input)) {
          throw new MCPError(
            ErrorType.VALIDATION_ERROR,
            `缺少必需字段: ${field}`,
            'MISSING_REQUIRED_FIELD',
            { field, schema }
          );
        }
      }
    }
  }
}
```

#### 高级日志系统

```typescript
// 结构化日志记录器
export enum LogLevel {
  DEBUG = 0,
  INFO = 1,
  WARN = 2,
  ERROR = 3
}

export class Logger {
  private level: LogLevel;
  private output: NodeJS.WritableStream;
  private context: string;

  constructor(context: string, level: LogLevel = LogLevel.INFO, output?: NodeJS.WritableStream) {
    this.context = context;
    this.level = level;
    this.output = output || process.stderr;
  }

  private log(level: LogLevel, message: string, meta?: any): void {
    if (level < this.level) {
      return;
    }

    const timestamp = new Date().toISOString();
    const levelName = LogLevel[level];
    
    const logEntry = {
      timestamp,
      level: levelName,
      context: this.context,
      message,
      meta,
      pid: process.pid
    };

    this.output.write(JSON.stringify(logEntry) + '\n');
  }

  debug(message: string, meta?: any): void {
    this.log(LogLevel.DEBUG, message, meta);
  }

  info(message: string, meta?: any): void {
    this.log(LogLevel.INFO, message, meta);
  }

  warn(message: string, meta?: any): void {
    this.log(LogLevel.WARN, message, meta);
  }

  error(message: string, error?: Error, meta?: any): void {
    const errorMeta = error ? {
      error: {
        name: error.name,
        message: error.message,
        stack: error.stack
      },
      ...meta
    } : meta;

    this.log(LogLevel.ERROR, message, errorMeta);
  }

  // 创建子记录器
  child(subContext: string): Logger {
    return new Logger(
      `${this.context}:${subContext}`,
      this.level,
      this.output
    );
  }
}

// 请求追踪
export class RequestTracker {
  private static requestId: number = 0;
  private logger: Logger;
  private startTime: bigint;
  public readonly id: string;

  constructor(operation: string, logger: Logger) {
    this.id = `req-${++RequestTracker.requestId}`;
    this.logger = logger.child(`${operation}[${this.id}]`);
    this.startTime = process.hrtime.bigint();
    
    this.logger.info('请求开始');
  }

  info(message: string, meta?: any): void {
    this.logger.info(message, { requestId: this.id, ...meta });
  }

  warn(message: string, meta?: any): void {
    this.logger.warn(message, { requestId: this.id, ...meta });
  }

  error(message: string, error?: Error, meta?: any): void {
    this.logger.error(message, error, { requestId: this.id, ...meta });
  }

  complete(result?: any): void {
    const duration = Number(process.hrtime.bigint() - this.startTime) / 1e6;
    this.logger.info('请求完成', {
      requestId: this.id,
      duration: `${duration.toFixed(2)}ms`,
      result: result ? '成功' : '失败'
    });
  }
}
```

## 最佳实践

### 代码质量和架构

#### 1. 模块化设计

```typescript
// 使用依赖注入实现松耦合
interface IStorage {
  get(key: string): Promise<any>;
  set(key: string, value: any): Promise<void>;
  delete(key: string): Promise<void>;
}

interface IValidator {
  validate(schema: any, data: any): boolean;
}

interface INotifier {
  notify(event: string, data: any): Promise<void>;
}

class MCPServer {
  constructor(
    private storage: IStorage,
    private validator: IValidator,
    private notifier: INotifier,
    private logger: Logger
  ) {}

  async handleToolCall(request: any): Promise<any> {
    const tracker = new RequestTracker('tool-call', this.logger);
    
    try {
      // 验证输入
      if (!this.validator.validate(request.schema, request.params)) {
        throw new MCPError(ErrorType.VALIDATION_ERROR, '输入验证失败');
      }

      // 处理请求
      const result = await this.processRequest(request);
      
      // 通知事件
      await this.notifier.notify('tool-called', {
        tool: request.params.name,
        success: true
      });

      tracker.complete(result);
      return result;
    } catch (error) {
      tracker.error('工具调用失败', error);
      await this.notifier.notify('tool-error', {
        tool: request.params.name,
        error: error.message
      });
      
      throw error;
    }
  }

  private async processRequest(request: any): Promise<any> {
    // 具体的请求处理逻辑
    return {};
  }
}
```

#### 2. 配置管理

```typescript
// 配置管理系统
export interface ServerConfig {
  server: {
    name: string;
    version: string;
    environment: string;
  };
  security: {
    allowedPaths: string[];
    maxFileSize: number;
    enableRateLimit: boolean;
    rateLimitWindow: number;
    rateLimitMax: number;
  };
  performance: {
    cacheSize: number;
    cacheTTL: number;
    batchSize: number;
    requestTimeout: number;
  };
  logging: {
    level: string;
    format: string;
    output: string;
  };
}

export class ConfigManager {
  private config: ServerConfig;
  private watchers: Array<(config: ServerConfig) => void> = [];

  constructor(configPath?: string) {
    this.config = this.loadConfig(configPath);
    this.watchConfigFile(configPath);
  }

  private loadConfig(configPath?: string): ServerConfig {
    const defaultConfig: ServerConfig = {
      server: {
        name: 'mcp-server',
        version: '1.0.0',
        environment: process.env.NODE_ENV || 'development'
      },
      security: {
        allowedPaths: [],
        maxFileSize: 10 * 1024 * 1024, // 10MB
        enableRateLimit: true,
        rateLimitWindow: 60000, // 1分钟
        rateLimitMax: 100
      },
      performance: {
        cacheSize: 1000,
        cacheTTL: 300000, // 5分钟
        batchSize: 10,
        requestTimeout: 30000 // 30秒
      },
      logging: {
        level: 'INFO',
        format: 'json',
        output: 'stderr'
      }
    };

    if (!configPath) {
      return defaultConfig;
    }

    try {
      const fileConfig = JSON.parse(fs.readFileSync(configPath, 'utf-8'));
      return this.mergeConfig(defaultConfig, fileConfig);
    } catch (error) {
      console.warn(`配置文件读取失败，使用默认配置: ${error.message}`);
      return defaultConfig;
    }
  }

  private mergeConfig(defaultConfig: any, fileConfig: any): any {
    // 深度合并配置对象
    function deepMerge(target: any, source: any): any {
      for (const key in source) {
        if (source[key] && typeof source[key] === 'object' && !Array.isArray(source[key])) {
          target[key] = target[key] || {};
          deepMerge(target[key], source[key]);
        } else {
          target[key] = source[key];
        }
      }
      return target;
    }

    return deepMerge({ ...defaultConfig }, fileConfig);
  }

  private watchConfigFile(configPath?: string): void {
    if (!configPath) return;

    fs.watchFile(configPath, (curr, prev) => {
      if (curr.mtime !== prev.mtime) {
        console.log('配置文件已更新，重新加载...');
        try {
          this.config = this.loadConfig(configPath);
          this.notifyWatchers();
        } catch (error) {
          console.error('配置重新加载失败:', error);
        }
      }
    });
  }

  private notifyWatchers(): void {
    this.watchers.forEach(watcher => {
      try {
        watcher(this.config);
      } catch (error) {
        console.error('配置监听器执行失败:', error);
      }
    });
  }

  get(): ServerConfig {
    return { ...this.config };
  }

  onChange(callback: (config: ServerConfig) => void): void {
    this.watchers.push(callback);
  }
}
```

### 安全性最佳实践

#### 1. 输入验证和清理

```typescript
// 输入清理和验证
export class InputSanitizer {
  static sanitizeFilePath(path: string): string {
    // 移除危险字符和路径遍历尝试
    return path
      .replace(/\.\./g, '') // 移除 ..
      .replace(/[<>:"|?*]/g, '') // 移除不安全字符
      .trim();
  }

  static sanitizeCommandInput(command: string): string {
    // 防止命令注入
    const dangerous = [';', '&&', '||', '|', '`', '$', '(', ')'];
    let sanitized = command;
    
    dangerous.forEach(char => {
      sanitized = sanitized.replace(new RegExp(`\\${char}`, 'g'), '');
    });
    
    return sanitized.trim();
  }

  static validateJsonSchema(data: any, schema: any): boolean {
    // 这里可以使用 ajv 或其他 JSON Schema 验证库
    // 简化版本的验证
    if (schema.type === 'object' && typeof data !== 'object') {
      return false;
    }
    
    if (schema.required) {
      for (const field of schema.required) {
        if (!(field in data)) {
          return false;
        }
      }
    }
    
    return true;
  }

  static limitStringLength(str: string, maxLength: number): string {
    if (str.length > maxLength) {
      return str.substring(0, maxLength - 3) + '...';
    }
    return str;
  }
}

// 速率限制
export class RateLimiter {
  private requests: Map<string, number[]> = new Map();
  private readonly windowMs: number;
  private readonly maxRequests: number;

  constructor(windowMs: number, maxRequests: number) {
    this.windowMs = windowMs;
    this.maxRequests = maxRequests;
    
    // 定期清理过期的记录
    setInterval(() => this.cleanup(), this.windowMs);
  }

  isAllowed(identifier: string): boolean {
    const now = Date.now();
    const requests = this.requests.get(identifier) || [];
    
    // 移除窗口外的请求
    const validRequests = requests.filter(time => now - time < this.windowMs);
    
    if (validRequests.length >= this.maxRequests) {
      return false;
    }
    
    // 添加当前请求
    validRequests.push(now);
    this.requests.set(identifier, validRequests);
    
    return true;
  }

  getRemainingRequests(identifier: string): number {
    const now = Date.now();
    const requests = this.requests.get(identifier) || [];
    const validRequests = requests.filter(time => now - time < this.windowMs);
    
    return Math.max(0, this.maxRequests - validRequests.length);
  }

  private cleanup(): void {
    const now = Date.now();
    
    for (const [identifier, requests] of this.requests.entries()) {
      const validRequests = requests.filter(time => now - time < this.windowMs);
      
      if (validRequests.length === 0) {
        this.requests.delete(identifier);
      } else {
        this.requests.set(identifier, validRequests);
      }
    }
  }
}
```

#### 2. 沙箱执行环境

```typescript
// 安全的命令执行环境
import { spawn, ChildProcess } from 'child_process';

export class SecureExecutor {
  private readonly allowedCommands: string[];
  private readonly maxExecutionTime: number;
  private readonly maxOutputSize: number;

  constructor(config: {
    allowedCommands: string[];
    maxExecutionTime: number;
    maxOutputSize: number;
  }) {
    this.allowedCommands = config.allowedCommands;
    this.maxExecutionTime = config.maxExecutionTime;
    this.maxOutputSize = config.maxOutputSize;
  }

  async execute(command: string, args: string[], workingDir?: string): Promise<{
    stdout: string;
    stderr: string;
    exitCode: number;
    executionTime: number;
  }> {
    const startTime = Date.now();
    
    // 验证命令是否被允许
    if (!this.allowedCommands.includes(command)) {
      throw new MCPError(
        ErrorType.PERMISSION_DENIED,
        `命令不被允许: ${command}`,
        'COMMAND_NOT_ALLOWED'
      );
    }

    return new Promise((resolve, reject) => {
      const child: ChildProcess = spawn(command, args, {
        cwd: workingDir,
        stdio: ['pipe', 'pipe', 'pipe'],
        timeout: this.maxExecutionTime,
        killSignal: 'SIGTERM'
      });

      let stdout = '';
      let stderr = '';
      let outputSize = 0;

      // 收集输出
      child.stdout?.on('data', (data) => {
        outputSize += data.length;
        if (outputSize > this.maxOutputSize) {
          child.kill('SIGTERM');
          reject(new MCPError(
            ErrorType.INTERNAL_SERVER_ERROR,
            '输出大小超过限制',
            'OUTPUT_SIZE_EXCEEDED'
          ));
          return;
        }
        stdout += data.toString();
      });

      child.stderr?.on('data', (data) => {
        outputSize += data.length;
        if (outputSize > this.maxOutputSize) {
          child.kill('SIGTERM');
          reject(new MCPError(
            ErrorType.INTERNAL_SERVER_ERROR,
            '输出大小超过限制',
            'OUTPUT_SIZE_EXCEEDED'
          ));
          return;
        }
        stderr += data.toString();
      });

      // 处理完成
      child.on('close', (code) => {
        const executionTime = Date.now() - startTime;
        resolve({
          stdout: stdout.trim(),
          stderr: stderr.trim(),
          exitCode: code || 0,
          executionTime
        });
      });

      // 处理错误
      child.on('error', (error) => {
        reject(new MCPError(
          ErrorType.INTERNAL_SERVER_ERROR,
          `命令执行失败: ${error.message}`,
          'EXECUTION_FAILED'
        ));
      });

      // 超时处理
      child.on('timeout', () => {
        child.kill('SIGKILL');
        reject(new MCPError(
          ErrorType.INTERNAL_SERVER_ERROR,
          '命令执行超时',
          'EXECUTION_TIMEOUT'
        ));
      });
    });
  }
}
```

### 测试策略

#### 1. 单元测试

```typescript
// 测试工具类
import { describe, it, expect, beforeEach, afterEach } from '@jest/globals';

describe('FileOperations', () => {
  let fileOps: FileOperations;
  let tempDir: string;

  beforeEach(async () => {
    tempDir = await fs.mkdtemp(path.join(os.tmpdir(), 'mcp-test-'));
    fileOps = new FileOperations([tempDir]);
  });

  afterEach(async () => {
    await fs.rmdir(tempDir, { recursive: true });
  });

  describe('readFile', () => {
    it('应该成功读取存在的文件', async () => {
      const testFile = path.join(tempDir, 'test.txt');
      const content = 'Hello, World!';
      await fs.writeFile(testFile, content);

      const result = await fileOps.readFile(testFile);

      expect(result.success).toBe(true);
      expect(result.data?.content).toBe(content);
    });

    it('应该拒绝访问不允许的路径', async () => {
      const unauthorizedFile = '/etc/passwd';

      const result = await fileOps.readFile(unauthorizedFile);

      expect(result.success).toBe(false);
      expect(result.message).toContain('访问被拒绝');
    });

    it('应该处理不存在的文件', async () => {
      const nonExistentFile = path.join(tempDir, 'nonexistent.txt');

      const result = await fileOps.readFile(nonExistentFile);

      expect(result.success).toBe(false);
      expect(result.message).toContain('失败');
    });
  });

  describe('writeFile', () => {
    it('应该成功写入文件', async () => {
      const testFile = path.join(tempDir, 'write-test.txt');
      const content = 'Test content';

      const result = await fileOps.writeFile(testFile, content);

      expect(result.success).toBe(true);
      
      const writtenContent = await fs.readFile(testFile, 'utf-8');
      expect(writtenContent).toBe(content);
    });

    it('应该创建备份文件', async () => {
      const testFile = path.join(tempDir, 'backup-test.txt');
      const originalContent = 'Original content';
      const newContent = 'New content';

      // 创建原始文件
      await fs.writeFile(testFile, originalContent);
      await new Promise(resolve => setTimeout(resolve, 10)); // 确保时间戳不同

      // 写入新内容并启用备份
      const result = await fileOps.writeFile(testFile, newContent, { backup: true });

      expect(result.success).toBe(true);

      // 检查原始文件是否更新
      const updatedContent = await fs.readFile(testFile, 'utf-8');
      expect(updatedContent).toBe(newContent);

      // 检查备份文件是否存在
      const files = await fs.readdir(tempDir);
      const backupFiles = files.filter(f => f.includes('.backup.'));
      expect(backupFiles.length).toBe(1);
    });
  });
});
```

#### 2. 集成测试

```typescript
// MCP 服务器集成测试
describe('MCP Server Integration', () => {
  let server: Server;
  let transport: MockTransport;

  beforeEach(async () => {
    transport = new MockTransport();
    server = await createTestServer();
    await server.connect(transport);
  });

  afterEach(async () => {
    await server.close();
  });

  it('应该正确处理工具列表请求', async () => {
    const request = {
      jsonrpc: '2.0',
      id: 1,
      method: 'tools/list'
    };

    const response = await transport.sendRequest(request);

    expect(response.result).toBeDefined();
    expect(response.result.tools).toBeInstanceOf(Array);
    expect(response.result.tools.length).toBeGreaterThan(0);
  });

  it('应该正确处理工具调用', async () => {
    const request = {
      jsonrpc: '2.0',
      id: 2,
      method: 'tools/call',
      params: {
        name: 'calculator',
        arguments: {
          operation: 'add',
          a: 2,
          b: 3
        }
      }
    };

    const response = await transport.sendRequest(request);

    expect(response.result).toBeDefined();
    expect(response.result.content).toBeInstanceOf(Array);
    expect(response.result.content[0].text).toContain('5');
  });

  it('应该处理无效的工具调用', async () => {
    const request = {
      jsonrpc: '2.0',
      id: 3,
      method: 'tools/call',
      params: {
        name: 'nonexistent-tool',
        arguments: {}
      }
    };

    const response = await transport.sendRequest(request);

    expect(response.error).toBeDefined();
    expect(response.error.message).toContain('未知工具');
  });
});

// Mock 传输层
class MockTransport {
  private handlers: Map<string, Function> = new Map();

  async sendRequest(request: any): Promise<any> {
    const handler = this.handlers.get(request.method);
    if (!handler) {
      return {
        jsonrpc: '2.0',
        id: request.id,
        error: {
          code: -32601,
          message: 'Method not found'
        }
      };
    }

    try {
      const result = await handler(request);
      return {
        jsonrpc: '2.0',
        id: request.id,
        result
      };
    } catch (error) {
      return {
        jsonrpc: '2.0',
        id: request.id,
        error: {
          code: -32603,
          message: error instanceof Error ? error.message : 'Internal error'
        }
      };
    }
  }

  setHandler(method: string, handler: Function): void {
    this.handlers.set(method, handler);
  }
}
```

### 部署和监控

#### 1. 生产环境配置

```typescript
// 生产环境启动脚本
#!/usr/bin/env node

import cluster from 'cluster';
import os from 'os';

const numCPUs = os.cpus().length;

if (cluster.isPrimary) {
  console.log(`主进程 ${process.pid} 正在启动`);

  // 启动工作进程
  for (let i = 0; i < Math.min(numCPUs, 4); i++) {
    cluster.fork();
  }

  // 监听工作进程退出
  cluster.on('exit', (worker, code, signal) => {
    console.log(`工作进程 ${worker.process.pid} 退出，代码: ${code}, 信号: ${signal}`);
    
    // 重启工作进程
    if (!worker.exitedAfterDisconnect) {
      console.log('重启工作进程...');
      cluster.fork();
    }
  });

  // 优雅关闭
  process.on('SIGTERM', () => {
    console.log('收到 SIGTERM，开始优雅关闭...');
    
    for (const worker of Object.values(cluster.workers || {})) {
      worker?.disconnect();
    }
    
    setTimeout(() => {
      console.log('强制关闭...');
      process.exit(0);
    }, 10000);
  });

} else {
  // 工作进程
  import('./server.js').then(({ startServer }) => {
    startServer().catch((error) => {
      console.error('服务器启动失败:', error);
      process.exit(1);
    });
  });
}
```

#### 2. 健康检查和监控

```typescript
// 健康检查端点
export class HealthChecker {
  private checks: Map<string, () => Promise<boolean>> = new Map();
  private lastResults: Map<string, { status: boolean; timestamp: number }> = new Map();

  addCheck(name: string, check: () => Promise<boolean>): void {
    this.checks.set(name, check);
  }

  async runChecks(): Promise<{ healthy: boolean; checks: any }> {
    const results: any = {};
    let overallHealthy = true;

    for (const [name, check] of this.checks.entries()) {
      try {
        const status = await Promise.race([
          check(),
          new Promise<boolean>((_, reject) => 
            setTimeout(() => reject(new Error('Timeout')), 5000)
          )
        ]);

        results[name] = {
          status: status ? 'healthy' : 'unhealthy',
          timestamp: Date.now()
        };

        this.lastResults.set(name, { status, timestamp: Date.now() });
        
        if (!status) {
          overallHealthy = false;
        }
      } catch (error) {
        results[name] = {
          status: 'error',
          error: error instanceof Error ? error.message : 'Unknown error',
          timestamp: Date.now()
        };
        
        this.lastResults.set(name, { status: false, timestamp: Date.now() });
        overallHealthy = false;
      }
    }

    return { healthy: overallHealthy, checks: results };
  }

  getLastResults(): any {
    const results: any = {};
    for (const [name, result] of this.lastResults.entries()) {
      results[name] = result;
    }
    return results;
  }
}

// 在服务器中使用健康检查
const healthChecker = new HealthChecker();

// 添加各种健康检查
healthChecker.addCheck('database', async () => {
  // 检查数据库连接
  try {
    await db.query('SELECT 1');
    return true;
  } catch {
    return false;
  }
});

healthChecker.addCheck('file_system', async () => {
  // 检查文件系统访问
  try {
    await fs.access('/tmp', fs.constants.W_OK);
    return true;
  } catch {
    return false;
  }
});

healthChecker.addCheck('memory', async () => {
  // 检查内存使用
  const usage = process.memoryUsage();
  const threshold = 1024 * 1024 * 1024; // 1GB
  return usage.heapUsed < threshold;
});
```

## 总结

通过遵循这些最佳实践和故障排除指南，你可以构建出稳定、安全、高性能的 MCP 服务器：

### 关键要点

1. **安全第一**：始终验证输入，限制访问权限，使用沙箱执行
2. **错误处理**：实现全面的错误处理和日志记录
3. **性能优化**：使用缓存、批处理和异步操作
4. **代码质量**：模块化设计，全面测试，清晰文档
5. **监控运维**：健康检查，性能监控，优雅关闭

### 持续改进

- 定期审查和更新安全策略
- 监控性能指标并持续优化
- 收集用户反馈并迭代改进
- 保持对 MCP 规范更新的关注

通过本教程的学习，你现在已经具备了创建专业级 MCP 服务器的所有知识。记住，好的软件是逐步迭代出来的，从简单开始，逐步添加功能和优化。

---

## 进一步学习资源

- [MCP 官方文档](https://modelcontextprotocol.io/)
- [Node.js 最佳实践](https://github.com/goldbergyoni/nodebestpractices)
- [TypeScript 手册](https://www.typescriptlang.org/docs/)
- [JSON-RPC 2.0 规范](https://www.jsonrpc.org/specification)