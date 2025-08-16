# 使用现有 MCP 服务器

在这一章中，我们将学习如何安装、配置和使用现有的 MCP 服务器。这是学习 MCP 的最佳起点，因为你可以立即体验到 MCP 的强大功能。

## 环境准备

### 安装 Node.js

大多数 MCP 服务器都基于 Node.js 构建，所以我们首先需要安装 Node.js。

**为什么需要 Node.js？**
Node.js 是 JavaScript 的运行环境，MCP 的官方实现和大多数社区服务器都使用 Node.js 开发，因为它：
- 有丰富的包生态系统 (npm)
- 天然支持 JSON 和异步操作
- 轻量级且易于部署

```bash
# 在 macOS 上使用 Homebrew 安装
brew install node

# 在 Ubuntu/Debian 上安装
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# 在 Windows 上，从官网下载安装包
# https://nodejs.org/
```

验证安装：
```bash
node --version  # 应该显示 v18 或更高版本
npm --version   # 显示 npm 版本
```

### 安装 MCP 客户端

目前最容易使用 MCP 的方式是通过 Claude Desktop 应用。

**为什么选择 Claude Desktop？**
- 官方支持 MCP 协议
- 图形化界面，适合初学者
- 配置简单，无需编程经验
- 可以立即看到 MCP 的效果

下载和安装：
1. 访问 [Claude 官网](https://claude.ai/download)
2. 下载适合你操作系统的版本
3. 安装应用程序

## 配置 Claude Desktop 使用 MCP

Claude Desktop 通过配置文件来管理 MCP 服务器连接。

### 找到配置文件位置

配置文件的位置因操作系统而异：

```bash
# macOS
~/Library/Application Support/Claude/claude_desktop_config.json

# Windows
%APPDATA%\Claude\claude_desktop_config.json

# Linux
~/.config/Claude/claude_desktop_config.json
```

### 创建基础配置文件

如果配置文件不存在，我们需要创建一个：

```json
{
  "mcpServers": {}
}
```

**配置文件结构解释：**
- `mcpServers`: 这是一个对象，包含所有 MCP 服务器的配置
- 每个服务器都有一个唯一的名称作为键
- 每个服务器配置包含命令、参数和环境变量

## 安装和配置第一个 MCP 服务器

让我们从一个简单的文件系统 MCP 服务器开始。

### 安装 filesystem MCP 服务器

```bash
# 全局安装 MCP 文件系统服务器
npm install -g @modelcontextprotocol/server-filesystem
```

**这个服务器能做什么？**
- 读取文件内容
- 列出目录内容
- 搜索文件
- 获取文件信息

### 配置 filesystem 服务器

编辑 Claude Desktop 配置文件，添加 filesystem 服务器：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "@modelcontextprotocol/server-filesystem",
        "/Users/your-username/Documents"
      ]
    }
  }
}
```

**配置参数解释：**
- `"filesystem"`: 服务器的名称，可以自定义
- `"command"`: 启动服务器的命令，这里使用 `npx` 来运行 npm 包
- `"args"`: 传递给命令的参数数组
  - 第一个参数是服务器包名
  - 第二个参数是允许访问的目录路径（**安全限制**）

**安全注意事项：**
路径参数定义了 MCP 服务器能够访问的目录范围。这是一个重要的安全机制，确保 AI 只能访问你明确授权的文件和目录。

### 重启 Claude Desktop

配置完成后，需要重启 Claude Desktop 应用程序以加载新的配置。

### 测试 filesystem 服务器

重启后，在 Claude Desktop 中测试：

```
你好！请帮我列出我的 Documents 目录中的所有文件。
```

如果配置正确，Claude 应该能够：
1. 识别到可用的文件系统工具
2. 调用相应的工具来列出文件
3. 返回目录内容

## 更多实用的 MCP 服务器

### Git 操作服务器

Git 服务器让 AI 能够执行版本控制操作。

```bash
# 安装 Git MCP 服务器
npm install -g @modelcontextprotocol/server-git
```

配置：
```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "@modelcontextprotocol/server-filesystem",
        "/Users/your-username/Documents"
      ]
    },
    "git": {
      "command": "npx",
      "args": [
        "@modelcontextprotocol/server-git",
        "--repository",
        "/Users/your-username/my-project"
      ]
    }
  }
}
```

**Git 服务器功能：**
- 查看提交历史
- 检查文件状态
- 创建分支
- 比较差异

**使用示例：**
```
请查看我的项目的最近 5 个提交记录，并总结主要的变更内容。
```

### SQLite 数据库服务器

数据库服务器允许 AI 查询和分析数据库内容。

```bash
# 安装 SQLite MCP 服务器
npm install -g @modelcontextprotocol/server-sqlite
```

配置：
```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "@modelcontextprotocol/server-filesystem",
        "/Users/your-username/Documents"
      ]
    },
    "git": {
      "command": "npx",
      "args": [
        "@modelcontextprotocol/server-git",
        "--repository",
        "/Users/your-username/my-project"
      ]
    },
    "sqlite": {
      "command": "npx",
      "args": [
        "@modelcontextprotocol/server-sqlite",
        "/path/to/your/database.db"
      ]
    }
  }
}
```

**SQLite 服务器功能：**
- 执行 SELECT 查询
- 查看表结构
- 分析数据分布
- 生成数据报告

**使用示例：**
```
请查看数据库中有哪些表，然后分析用户表中的用户分布情况。
```

### Web 搜索服务器

Web 搜索服务器让 AI 能够搜索网络内容。

```bash
# 安装 Web 搜索 MCP 服务器
npm install -g @modelcontextprotocol/server-brave-search
```

配置：
```json
{
  "mcpServers": {
    "web-search": {
      "command": "npx",
      "args": [
        "@modelcontextprotocol/server-brave-search"
      ],
      "env": {
        "BRAVE_API_KEY": "your-brave-api-key-here"
      }
    }
  }
}
```

**获取 Brave API 密钥：**
1. 访问 [Brave Search API](https://api.search.brave.com/)
2. 注册账户并获取 API 密钥
3. 将密钥添加到环境变量中

**Web 搜索功能：**
- 搜索最新信息
- 获取网页内容
- 查找特定主题的资源

## 高级配置技巧

### 环境变量管理

对于需要 API 密钥的服务器，建议使用环境变量：

```json
{
  "mcpServers": {
    "web-search": {
      "command": "npx",
      "args": ["@modelcontextprotocol/server-brave-search"],
      "env": {
        "BRAVE_API_KEY": "your-api-key"
      }
    }
  }
}
```

**最佳实践：**
- 不要在配置文件中硬编码敏感信息
- 使用系统环境变量或配置文件管理密钥
- 定期轮换 API 密钥

### 多服务器协同工作

你可以同时配置多个 MCP 服务器，让 AI 能够协调使用不同的功能：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "@modelcontextprotocol/server-filesystem",
        "/Users/your-username/projects"
      ]
    },
    "git": {
      "command": "npx",
      "args": [
        "@modelcontextprotocol/server-git",
        "--repository",
        "/Users/your-username/projects/my-app"
      ]
    },
    "sqlite": {
      "command": "npx",
      "args": [
        "@modelcontextprotocol/server-sqlite",
        "/Users/your-username/projects/my-app/database.db"
      ]
    }
  }
}
```

**协同工作示例：**
```
请分析我的项目：
1. 查看最近的代码变更
2. 读取数据库中的用户数据
3. 检查配置文件中的设置
4. 给出项目状态的综合报告
```

## 常见使用场景

### 1. 代码项目分析

**场景描述：**你想让 AI 帮助分析一个代码项目的结构和问题。

**配置需求：**
- filesystem 服务器（读取代码文件）
- git 服务器（查看版本历史）

**示例对话：**
```
用户：请分析我的 React 项目，查看代码结构是否合理，有什么可以改进的地方？

AI：我来帮你分析项目结构。首先让我查看项目文件：
[调用 filesystem 工具列出项目文件]

然后查看最近的提交历史：
[调用 git 工具查看提交记录]

基于分析，我发现以下几个可以改进的地方：
1. 组件文件可以更好地组织到子目录中
2. 某些文件缺少适当的注释
3. 建议添加 TypeScript 类型定义
```

### 2. 数据分析和报告

**场景描述：**你有一个包含业务数据的 SQLite 数据库，想要生成分析报告。

**配置需求：**
- sqlite 服务器（查询数据库）
- filesystem 服务器（保存报告）

**示例对话：**
```
用户：请分析销售数据库，生成本月的销售报告。

AI：我来分析销售数据并生成报告：

1. 首先查看数据库结构：
[调用 sqlite 工具查看表结构]

2. 分析本月销售数据：
[调用 sqlite 工具查询销售数据]

3. 生成详细报告：
- 总销售额：¥50,000
- 最佳销售产品：产品A
- 销售趋势：上涨15%

是否需要我将这个报告保存到文件中？
```

### 3. 学习和研究

**场景描述：**你想研究某个技术主题，需要查找最新信息和本地资料。

**配置需求：**
- web-search 服务器（搜索最新信息）
- filesystem 服务器（访问本地笔记）

**示例对话：**
```
用户：我想学习 Kubernetes，请帮我整理一个学习计划。

AI：我来帮你制定 Kubernetes 学习计划：

1. 首先搜索最新的 Kubernetes 资源：
[调用 web-search 工具搜索相关信息]

2. 查看你的本地学习笔记：
[调用 filesystem 工具查看现有笔记]

3. 基于搜索结果和你的背景，我建议以下学习路径：
- 第1周：基础概念和容器技术
- 第2周：Kubernetes 架构和组件
- 第3周：实践部署和配置
...
```

## 故障排除

### 服务器无法启动

**问题症状：**Claude Desktop 中看不到 MCP 服务器提供的工具。

**排查步骤：**

1. **检查配置文件语法：**
   ```bash
   # 验证 JSON 语法
   cat ~/Library/Application\ Support/Claude/claude_desktop_config.json | python -m json.tool
   ```

2. **检查命令路径：**
   ```bash
   # 确认 npm 包已安装
   npm list -g @modelcontextprotocol/server-filesystem
   
   # 测试命令是否可执行
   npx @modelcontextprotocol/server-filesystem --help
   ```

3. **查看 Claude Desktop 日志：**
   - macOS: `~/Library/Logs/Claude/`
   - Windows: `%APPDATA%\Claude\logs\`
   - Linux: `~/.config/Claude/logs/`

### 权限问题

**问题症状：**MCP 服务器启动了，但无法访问文件或目录。

**解决方案：**

1. **检查路径权限：**
   ```bash
   # 确认目录存在且有读取权限
   ls -la /path/to/directory
   ```

2. **使用绝对路径：**
   ```json
   {
     "mcpServers": {
       "filesystem": {
         "command": "npx",
         "args": [
           "@modelcontextprotocol/server-filesystem",
           "/Users/username/Documents"  // 使用绝对路径
         ]
       }
     }
   }
   ```

### API 密钥问题

**问题症状：**Web 搜索或其他需要 API 的服务无法工作。

**解决方案：**

1. **验证 API 密钥：**
   ```bash
   # 测试 API 密钥是否有效
   curl -H "X-Subscription-Token: your-api-key" \
        "https://api.search.brave.com/res/v1/web/search?q=test"
   ```

2. **检查环境变量设置：**
   ```json
   {
     "mcpServers": {
       "web-search": {
         "command": "npx",
         "args": ["@modelcontextprotocol/server-brave-search"],
         "env": {
           "BRAVE_API_KEY": "your-actual-api-key-here"
         }
       }
     }
   }
   ```

## 下一步

现在你已经学会了如何使用现有的 MCP 服务器，你可以：

1. 尝试组合使用多个服务器完成复杂任务
2. 探索更多社区开发的 MCP 服务器
3. 开始学习如何创建自己的 MCP 服务器

在下一章中，我们将学习如何从零开始创建一个自定义的 MCP 服务器，让你能够为特定需求开发专门的功能。

---

## 参考资源

- [MCP 官方服务器列表](https://github.com/modelcontextprotocol/servers)
- [Claude Desktop 配置指南](https://docs.anthropic.com/en/docs/build-with-claude/computer-use)
- [MCP 社区服务器](https://github.com/topics/model-context-protocol)