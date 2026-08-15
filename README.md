# codex-gatekeeper-710
项目定位与核心功能 当 AI Agent 生态越来越丰富时，第三方技能的安全风险也随之而来。codex-gatekeeper 的目标是成为一个防御性审查工具，在技能安装前或执行敏感操作前进行静态扫描
功能模块	说明
技能安全扫描	离线扫描 Codex 技能目录，识别提示注入、危险系统调用、数据外泄等 16+ 类风险
沙盒策略检查	验证 AGENTS.md 和 .codex/config.toml 中的权限配置是否合理（如 sandbox_mode 与 approval_policy 的组合）
供应链审查	检查依赖来源是否可信、版本是否锁定、是否存在拼写欺诈风险
审计报告生成	输出结构化的安全报告，包含证据、风险等级和修复建议
CI 集成	提供 JSON/SARIF 输出，可接入 GitHub Actions 等 CI 流水线
项目目录结构
text
codex-gatekeeper/
├── README.md                     # 项目介绍、安装、使用说明
├── AGENTS.md                     # Codex 的"跨 Agent 脊柱"——定义行为规则和安全边界[citation:1]
├── .codex/
│   └── config.toml               # Codex 沙盒与审批策略配置[citation:6]
├── skills/
│   └── gatekeeper-scan/          # 核心扫描技能，Codex 可调用
│       ├── SKILL.md              # 技能定义（触发条件、工作流、输出格式）[citation:8]
│       └── references/
│           ├── risk-taxonomy.md  # 风险分类与严重程度定义[citation:5]
│           └── report-template.md # 审计报告模板
├── scripts/
│   ├── scan_skill.py             # Python 实现的静态扫描脚本（离线只读）[citation:5][citation:12]
│   └── validate_config.mjs       # 验证 Codex 配置文件安全性的 Node.js 脚本[citation:1]
├── .github/
│   └── workflows/
│       └── security-scan.yml     # CI 示例：每次 PR 自动扫描技能变更
└── tests/
    └── test_scan_skill.py        # 扫描脚本单元测试
关键文件内容（部分示例）
AGENTS.md —— 定义安全边界
markdown
# AGENTS.md

## 项目概览
codex-gatekeeper 是一个为 Codex 提供安全审计能力的工具集，旨在对第三方技能进行防御性审查。

## 安全边界
- **敏感路径保护**：`.codex/`、`scripts/` 等目录下的配置变更需显式审批
- **离线优先**：扫描过程不联网、不执行目标技能的任何脚本（静态只读）[citation:2]
- **沙盒约束**：Codex 自身在 `workspace-write + on-request` 模式下运行[citation:6]

## 测试要求
- 所有 PR 必须通过安全扫描脚本的单元测试
- 修改 `references/` 下的风险规则时，需附带示例验证

## 代码规范
- Python 脚本优先考虑可读性和离线可用性
- 避免引入不必要的第三方依赖
.codex/config.toml —— Codex 权限策略
toml
# Codex 沙盒模式配置
sandbox_mode = "workspace-write"
approval_policy = "on-request"
approvals_reviewer = "user"

# 规则：对指定路径的操作触发额外审批
[[rules]]
path = ".codex/"
action = "write"
message = "修改 Codex 配置需要额外确认"
skills/gatekeeper-scan/SKILL.md —— 技能定义
markdown
---
name: gatekeeper-scan
description: 安全审计 Codex 技能目录，识别风险并生成报告。当用户要求审查技能安全性时触发。
---

## 工作流
1. 识别目标技能目录（或默认扫描 `~/.codex/skills/` 下所有技能）
2. 运行 `scripts/scan_skill.py --path <target> --format json`
3. 解析扫描结果，按严重程度分类（Critical / High / Medium / Low）
4. 生成审计报告，包含证据、风险说明和修复建议

## 输出格式
**安全决策**：通过 / 修复后通过 / 不建议安装
**风险等级**：低 / 中 / 高 / 严重

**确认发现**
- [严重] 问题标题
  证据：文件路径:行号
  风险：具体影响描述
  修复：具体建议

**需人工复核**
- [可疑模式] 描述及其位置
快速上手指南
1. 创建仓库并初始化
bash
mkdir codex-gatekeeper && cd codex-gatekeeper
git init
# 按上述结构创建文件和目录
2. 复制关键内容
将上面给出的 AGENTS.md、.codex/config.toml、SKILL.md 内容填入对应文件。

3. 添加扫描脚本（简化版）
创建 scripts/scan_skill.py，实现一个基础的静态扫描函数，检查：

文件中的危险关键词（eval、exec、subprocess、curl、wget）

敏感模式（API Key、Token 正则匹配）

网络请求相关的字符串

4. 提交到 GitHub
bash
git add .
git commit -m "feat: init codex-gatekeeper security audit project"
git remote add origin https://github.com/你的用户名/codex-gatekeeper.git
git push -u origin main
进阶：参考现有成熟项目
如果你想更快上手或借鉴更多实现细节，可以关注以下几个已有项目：

codex-copilot-agent-settings-for-vscode：提供了 AGENTS.md 和配置校验脚本的完整模板

skill-inspector-ps：PowerShell 实现的离线安全扫描器，覆盖 16+ 类风险，包含简报模式和已审记录功能

audit-skill-security：Python 实现的技能审计工具，包含风险分类和报告模板

这个项目既有实际的代码安全审查功能（扫描脚本），又体现了对 AI Agent 生态安全的前瞻性思考（为 Codex 建立安全护栏）。如果你有特定的扫描规则或使用场景想调整，可以进一步细化 references/risk-taxonomy.md 中的风险分类和检查点。
