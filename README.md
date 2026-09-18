# CyberSec-AI Agent 使用手册 v0.6.0

## 一、概述

CyberSec-AI Agent 是企业级网络安全智能体系统，集成 **7 个专业安全 Agent**（威胁情报、漏洞扫描、威胁狩猎、应急响应、渗透测试、报告生成、风险评估），基于 LLM 驱动，具备意图解析、任务编排、反馈闭环和自我学习能力。

```
Agent = 推理能力 + 执行能力 + 结果闭环 + 自我学习
```

---

## 二、安装与运行

### 2.1 Windows 可执行文件（推荐）

```bash
# 查看版本
cybersec.exe --version

# 中文显示需设置 UTF-8 编码
PYTHONIOENCODING=utf-8 cybersec.exe --help
```

### 2.2 Python 安装（源码开发）

```bash
pip install -e .
python -m cybersec_agent.cli --help
```

### 2.3 环境变量

```bash
export OPENAI_API_KEY="sk-..."          # OpenAI
export OPENROUTER_API_KEY="sk-or-..."   # OpenRouter
export ANTHROPIC_API_KEY="sk-ant-..."   # Anthropic（需 pip install anthropic）
```

---

## 三、CLI 命令速查

```
cybersec --help
├── init [-v]          启动横幅 + 合规声明 + 配置概览
├── agents [-j]        列出 7 个智能体及模型配置
├── stats [-j] [-a]    查看学习统计（可指定 Agent）
├── model              查看/切换默认模型
│   ├── --list         列出所有可用模型
│   └── --set <p/m>    切换，例: ollama/llama3.2
├── run <任务>         运行安全任务
│   ├── --role blue|red|purple
│   ├── --model <p/m>  临时覆盖模型
│   └── --verbose / -v
└── api                管理 API 密钥
    ├── set <p[/m]>    保存密钥到加密存储
    │   ├── --key <k>  命令行指定密钥
    │   └── --url <u>  自定义端点（custom 模式）
    ├── show           查看各提供商密钥状态
    ├── clear <p>      移除已保存密钥
    └── test <p>       发送 PONG 请求验证密钥
```

---

## 四、API 密钥管理（核心）

密钥采用 **Fernet 加密**存储在 `~/.cybersec/keys.enc`，不再明文写入 config.json。

### 4.1 查看当前状态

```bash
cybersec api show
```

输出示例：
```
API Key 状态
────────────────────────────────────────
  ✓  openai        已保存
  ×  anthropic     未配置
  ○  openrouter    环境变量 OPENROUTER_API_KEY
  ✓  ollama        无需 Key（本地运行）
  ×  custom        自定义端点
```

图例：`✓` 已保存  `×` 未配置  `○` 环境变量

### 4.2 设置密钥

```bash
# 方式一：命令行指定（适合 CI/脚本）
cybersec api set openai -k "sk-proj-xxx"

# 方式二：交互式输入（适合终端）
cybersec api set openai

# 同时设置模型和端点
cybersec api set openai/gpt-4o-mini -k "sk-xxx"
cybersec api set custom/gpt-4o --url http://localhost:8000/v1 -k "sk-xxx"

# 从环境变量读取
export ANTHROPIC_API_KEY="sk-ant-xxx"
cybersec api set anthropic   # 自动检测环境变量
```

### 4.3 测试连通性

```bash
cybersec api test openai               # 使用默认模型 gpt-4o-mini
cybersec api test openai -m o3-mini    # 指定测试模型
cybersec api test ollama               # 仅提示检查服务状态
```

成功后显示：`[OK] 连接成功！响应: pong`

### 4.4 清除密钥

```bash
cybersec api clear openai   # 移除已保存的 openai key
```

---

## 五、模型配置

### 5.1 查看所有可用模型

```bash
cybersec model --list
```

| Provider | 推荐模型 | 说明 |
|----------|----------|------|
| openai | gpt-4o | 旗舰模型，综合能力最强 |
| openai | gpt-4o-mini | 轻量快速，性价比高 |
| openai | o3-mini | 强推理，适合复杂分析 |
| openrouter | anthropic/claude-3.5-sonnet | Claude 3.5 Sonnet |
| ollama | llama3.2 | 本地免费运行 |
| ollama | qwen2.5 | 阿里 Qwen，中文能力强 |
| custom | <your-model> | 任意 OpenAI 兼容端点 |

### 5.2 切换默认模型

```bash
# 切换到本地 Ollama（零成本）
cybersec model --set ollama/llama3.2

# 切换到 OpenAI 轻量模型
cybersec model --set openai/gpt-4o-mini

# 切换到自定义端点
cybersec model --set custom/gpt-4o --url http://localhost:8000/v1
```

### 5.3 以 JSON 输出

```bash
cybersec model --json    # 程序化读取当前配置
```

---

## 六、运行安全任务

### 6.1 基本用法

```bash
# 蓝队：漏洞扫描
cybersec run "对 192.168.1.0/24 进行漏洞扫描" --role blue

# 红队：渗透测试
cybersec run "对 example.com 进行渗透测试" --role red

# 紫队：应急响应
cybersec run "主机 xss 检测到 Mimikatz 进程，IOC=abc123" --role purple -v
```

### 6.2 角色说明

| 角色 | 适用场景 | 主要 Agent |
|------|----------|-----------|
| `blue` | SOC 防守、漏洞管理、威胁狩猎 | ThreatIntel、VulnScan、ThreatHunt、IncidentResp |
| `red` | 红队演练、攻击模拟、权限提升 | PenTest、Report |
| `purple` | 攻防协同、综合报告、风险评估 | 全部 Agent 协同 |

### 6.3 临时覆盖模型

```bash
# 单次任务使用不同模型
cybersec run "扫描内网段" --role blue --model ollama/llama3.2
```

---

## 七、智能体列表

```bash
cybersec agents
```

| Agent | 角色 | 核心能力 |
|-------|------|---------|
| threat_intel_agent | 🔵 蓝队 | IOC 聚合、ATT&CK 映射、APT 归因 |
| vuln_scan_agent | 🔵 蓝队 | 端口扫描、CVE 匹配、CVSS/EPSS 评分 |
| threat_hunt_agent | 🔵 蓝队 | UEBA 异常检测、Sigma/YARA 规则生成 |
| incident_response_agent | 🔵 蓝队 | 事件分级、取证、遏制、时间线重建 |
| pen_test_agent | 🔴 红队 | Recon→Scan→Exploit→Post-exploit |
| report_agent | 🟣 紫队 | Markdown/HTML 报告、等保 2.0 合规映射 |
| risk_assessment_agent | 🟣 紫队 | 资产价值×威胁×脆弱性 = 风险值 |

---

## 八、学习统计

```bash
# 总体统计
cybersec stats

# 指定 Agent
cybersec stats -a vuln_scan_agent

# JSON 输出
cybersec stats --json
```

系统记录每次任务的成功率、耗时、参数组合，自动注入优化建议。

---

## 九、首次使用流程

```bash
# 1. 初始化（显示合规声明 + 当前配置）
cybersec init

# 2. 配置 API 密钥
cybersec api set openai -k "sk-proj-xxx"

# 3. 测试连通性
cybersec api test openai

# 4. 查看智能体
cybersec agents

# 5. 运行第一个任务
cybersec run "扫描 10.0.0.0/24 并评估风险" --role blue -v
```

---

## 十、常见问题

### Q: 中文显示乱码？
```bash
# Windows CMD
chcp 65001 && cybersec.exe --help

# PowerShell（推荐）
[Console]::OutputEncoding = [Text.Encoding]::UTF8
.\cybersec.exe --help

# 或者临时设置
PYTHONIOENCODING=utf-8 cybersec.exe --help
```

### Q: `api test` 超时？
网络不通或 API Key 无效时可能超时。检查：
- 防火墙是否允许访问 `api.openai.com`
- API Key 是否正确（运行 `cybersec api test openai -m gpt-4o-mini`）

### Q: 密钥是否明文存储？
否。密钥加密存储在 `~/.cybersec/keys.enc`（Fernet AES-128-CBC），加密密钥基于机器指纹派生，其他用户/进程无法解密。config.json 不含任何密钥信息。

### Q: 如何切换到本地模型？
```bash
# 1. 安装 Ollama：https://ollama.com
# 2. 拉取模型
ollama pull llama3.2
# 3. 配置 cybersec
cybersec model --set ollama/llama3.2
```

### Q: 如何备份密钥？
```bash
# 加密文件本身需备份（含机器绑定密钥）
cp ~/.cybersec/keys.enc ~/backup/
```

---

## 十一、文件结构

```
~/.cybersec/
├── config.json      # 模型配置（provider/model/base_url），不含密钥
└── keys.enc         # Fernet 加密的 API 密钥存储

cybersec-agent/
├── cybersec_agent/
│   ├── cli.py            # CLI 入口
│   ├── main.py           # Orchestrator 类
│   ├── core/
│   │   ├── api_keys.py   # KeyManager（加密密钥管理）
│   │   ├── learning.py   # LearningEngine（学习引擎）
│   │   ├── llm.py        # LLMClient（统一 LLM 调用）
│   │   ├── perception.py # PerceptionEngine（意图解析）
│   │   └── base.py       # BaseAgent / Role / TaskStatus
│   ├── commands/
│   │   ├── api.py        # api set/show/clear/test
│   │   ├── model.py      # model --list/--set
│   │   ├── run.py        # run <任务>
│   │   ├── agents.py     # agents
│   │   ├── stats.py      # stats
│   │   └── init_cmd.py   # init
│   ├── agents/           # 7 个专业 Agent
│   └── config/
│       └── models.py     # ModelConfig / AgentModelConfig
├── build_exe.py          # PyInstaller 打包脚本
├── pyproject.toml
└── README.md
```

---

## 十二、合规声明

本系统严格遵循《中华人民共和国网络安全法》及相关法律法规。
所有安全评估、渗透测试及漏洞扫描操作**必须在获得明确书面授权后进行**。
禁止对未授权目标实施任何形式的破坏性攻击或未经授权的系统访问。

使用本系统即表示您确认已获得相应授权，并承诺仅用于合法合规的安全研究目的。
