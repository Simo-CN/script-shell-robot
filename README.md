# ScriptShell Robot (SSR)

一个基于 Electron + Python 的跨平台终端自动化工具，集成 AI 智能助手和二级安全审核机制，为网络设备运维、系统管理提供智能化支持。

## ✨ 核心特性

### 🤖 双模式 AI 智能助手

- **Ask 模式**：通用问答对话，支持上下文记忆、自动压缩和会话管理
- **Agent 模式**：自主 AI 助手，可直接操作脚本和终端
  - **工具调用能力**：`read_script`（读取脚本）、`edit_script`（编辑脚本）、`execute_command`（执行命令）、`read_terminal`（读取终端输出）
  - **自主决策**：AI 可根据任务需求自动调用工具完成复杂操作
  - **逐行执行**：每条命令独立发送，实时跟踪输出，支持分页符处理

### 🛡️ 二级安全审核机制

所有通过 AI Agent 执行的命令都会经过专用 AI 模型进行安全审核：

| 安全级别 | 说明 | 示例 | 审核策略 |
|---------|------|------|---------|
| **L0** - 功能级 | 仅切换视图，无数据读写 | `cd`, `interface GE0/0/0` | ✅ 自动通过 |
| **L1** - 读取级 | 查看非敏感信息 | `ls`, `pwd`, `whoami`, `uptime` | ✅ 自动通过 |
| **L2** - 敏感信息读取级 | 读取可能含敏感数据的内容 | `cat /etc/passwd`, `show running-config` | ⚠️ 自动通过（警告） |
| **L3** - 功能参数配置级 | 修改非核心服务参数 | 调整 SMTP 优先级、nginx 超时值 | ⏸️ **需人工确认** |
| **L4** - 底层配置级 | 修改核心网络/系统参数 | 配置 IP 地址、路由协议、防火墙规则 | ⏸️ **需人工确认** |
| **L5** - 物理配置级 | 影响物理层或导致服务中断 | `shutdown`, `reboot`, `interface shutdown` | ⏸️ **需人工确认** |

- **专用审核模型**：支持配置独立的 AI 模型用于命令审核，与对话模型解耦
- **上下文感知**：审核系统理解网络设备特性（如 Cisco/Huawei 命令差异）
- **风险分级**：safe / caution / confirm / danger 四级风险提示

### 🖥️ 多连接支持

- **SSH**：支持密钥认证、密码认证，集成 SFTP 远程文件管理
- **Telnet**：支持传统网络设备管理
- **Serial**：支持串口连接（如交换机 Console 口）
- **加密存储**：使用 Fernet 加密存储连接凭据和敏感信息

### 📜 智能脚本系统

- **动态参数**：使用 `$参数名$` 定义可复用脚本模板
- **逐行执行**：每行命令独立发送，保证顺序性和可追踪性
- **文件夹管理**：支持脚本分类、批量选择
- **AI 辅助编辑**：Agent 模式可根据需求自动编辑脚本

### 🎨 现代化界面

- **三栏式布局**：左侧脚本/连接列表、中间终端/编辑器标签页、右侧 AI 助手/参数面板
- **深色/浅色主题**：支持主题切换
- **终端仿真**：基于 xterm.js，完整支持 ANSI 转义、快捷键、特殊字符

## 🏗️ 技术架构

- **前端**: Electron 41+ + xterm.js + marked.js（Markdown 渲染）
- **后端**: Python 3.12+ (异步架构)
- **通信**: stdin/stdout JSON-RPC（前后端解耦）
- **数据库**: SQLite (aiosqlite) - 存储连接配置、脚本、AI 模型配置
- **AI 集成**: OpenAI SDK（兼容任意 OpenAI 兼容接口）
- **加密**: cryptography (Fernet) - 敏感数据加密存储
- **网络库**: paramiko (SSH/SFTP)、pyserial (Serial)

### 架构亮点

- **前后端完全解耦**：通过 JSON-RPC over stdio 通信，支持独立打包
- **异步 I/O**：Python 后端全异步设计，高效处理并发连接
- **流式响应**：AI 对话支持流式输出，实时反馈
- **上下文智能管理**：自动 token 计数、超限自动压缩、保留关键信息

## 📦 快速开始

### AI 模型配置

首次使用 AI 功能需要配置模型：

1. 在界面右侧点击 AI 助手面板
2. 点击"管理模型"按钮
3. 添加提供商（支持 OpenAI、Anthropic、Deepseek 等兼容接口）
4. 添加模型并设置 API Key
5. 推荐为审核功能单独配置一个快速模型（如 gpt-4o-mini）

## 📝 脚本编写规范

### 动态参数格式

使用 `$参数名$` 定义可复用参数：

```bash
# 网络设备配置备份脚本
terminal length 0
show running-config | redirect flash:/$hostname$_config_$backup_date$.cfg
dir flash:
```

```bash
# 服务器部署脚本
cd $deploy_path$
git pull origin $branch$
docker-compose -f $compose_file$ up -d
docker ps | grep $service_name$
```

### 参数命名规则

- ✅ 只能包含字母、数字、下划线：`$user_id$`、`$server_1$`
- ❌ 不能包含空格和特殊字符：`$user id$`、`$server-1$`

### 网络设备最佳实践

```bash
# 禁用分页（脚本开头）
terminal length 0
terminal width 200

# 查看配置
display current-configuration

# 备份配置
save force

# 查看接口状态
display interface brief
```

### AI Agent 执行机制

- **逐行发送**：脚本每一行作为独立命令发送到远程连接
- **实时输出**：每行执行后立即返回输出，便于 AI 决策
- **自动审核**：所有命令在执行前都会经过二级安全审核
- **分页处理**：AI 能识别 `--More--` 并自动发送空格继续

## 💡 开发说明

### 前后端通信协议

使用 JSON-RPC 风格通过 stdin/stdout 进行双向通信

### AI Agent 工具调用流程

1. **用户请求** → Agent 模式对话
2. **AI 决策** → 生成工具调用（如 `execute_command`）
3. **安全审核** → 专用 AI 模型评估命令风险等级（L0-L5）
4. **人工确认** → L3-L5 级别命令等待用户确认
5. **执行命令** → 发送到远程连接，实时返回输出
6. **AI 分析** → 根据输出继续决策或向用户报告

### 扩展开发

- **添加新工具**：在 [model_tools.py](backend/model_tools.py) 中定义函数和 OpenAI Function Calling 描述
- **自定义提示词**：修改 [prompts/](prompts/) 目录下的 Markdown 文件
- **修改审核规则**：编辑 [prompts/audit.md](prompts/audit.md) 调整分级标准

## 🚀 使用场景

- **网络设备运维**：批量配置备份、接口状态监控、路由表查询
- **服务器管理**：应用部署、日志分析、性能监控
- **自动化测试**：设备功能测试、压力测试、故障恢复验证
- **知识库构建**：AI Agent 可自动学习设备特性，辅助新手运维

## 📄 许可证

仅参考学习、交流使用，禁止商用
