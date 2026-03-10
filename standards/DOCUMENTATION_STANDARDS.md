# TraderMate 文档标准

**版本**: 1.0
**生效日期**: 2026-03-02
**维护者**: Writer (文档工程师)
**适用范围**: 所有项目文档（docs/ 及其子目录，KNOWLEDGE_BASE/）

---

## 1. 文档编写原则

### 1.1 核心原则
- **用户为中心**: 始终考虑读者是谁，他们需要什么
- **清晰第一**: 用简单语言表达复杂概念，避免行话
- **示例丰富**: 每个抽象概念配以具体示例
- **保持最新**: 过时文档比没有文档更糟
- **可操作**: 提供明确的步骤、命令和预期结果

### 1.2 文档"S.T.A.N.D."标准
| 字母 | 标准 | 说明 |
|------|------|------|
| **S** | **Structure** (结构清晰) | 使用标题层级、列表、表格组织内容 |
| **T** | **Truthful** (真实准确) | 验证所有信息、命令、代码示例 |
| **A** | **Actionable** (可执行) | 提供可复制粘贴的命令和配置 |
| **N** | **Necessary** (必要简洁) | 删除冗余，保留核心信息 |
| **D** | **Discoverable** (易于发现) | 良好的导航、链接、搜索 |

---

## 2. 文件命名和组织

### 2.1 文件名规范
- 使用 **kebab-case** (小写字母加连字符)
- 示例: `getting-started.md`, `api-reference.md`
- 禁止: 空格、大写字母、下划线、特殊字符

### 2.2 目录结构原则
- 按**受众**分组 (`user/`, `developer/`, `operations/`)
- 按**功能**分组 (`api/`, `frontend/`, `database/`)
- 深度不超过 3 层（`a/b/c/d.md` 是极限）

### 2.3 推荐结构（未来KNOWLEDGE_BASE/）
```
KNOWLEDGE_BASE/
├── README.md              # 入口和导航
├── user-guides/           # 用户指南
│   ├── getting-started.md
│   ├── strategies.md
│   └── troubleshooting.md
├── api-reference/         # API参考
│   ├── overview.md
│   ├── authentication.md
│   └── models.md
├── operations/            # 运维文档
│   ├── deployment.md
│   ├── monitoring.md
│   └── runbooks/
└── development/           # 开发者指南
    ├── contributing.md
    ├── testing.md
    └── architecture/
```

---

## 3. Markdown格式规范

### 3.1 标题层级
- `#` (H1) - 文档主标题（每文档唯一）
- `##` (H2) - 主要章节
- `###` (H3) - 子章节
- `####` (H4) - 深层嵌套（尽量少用）
- 不要跳级（不要从H1跳到H3）

### 3.2 列表
无序列表使用 `-`：
```markdown
- 项目一
- 项目二
  - 子项目（缩进4个空格或1个制表符）
```

有序列表使用 `1.`：
```markdown
1. 第一步
2. 第二步
   1. 子步骤（使用1.1.格式或嵌套列表）
```

### 3.3 代码块
- 指定语言以实现语法高亮：
  - `python` - Python代码
  - `bash` / `shell` / `sh` - Shell命令
  - `json` / `yaml` - 配置文件
  - `sql` - SQL语句
  - `typescript` / `tsx` - TypeScript代码
  - `markdown` - Markdown示例

示例：
````markdown
```bash
docker-compose up -d
```
````

### 3.4 强调
- **粗体**: `**text**` 或 `__text__` - 用于重要内容
- *斜体*: `*text*` 或 `_text_` - 用于术语定义
- `代码`: `` `text` `` - 用于文件、命令、变量
- ~~删除线~~: `~~text~~` - 用于已废弃内容（少用）

### 3.5 表格
使用GitHub Flavored Markdown表格：
```markdown
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | int | 是 | 策略ID |
| `name` | str | 是 | 策略名称 |
```

- 表头对齐使用 `|:---|` (左), `|:---:|` (中), `|---:|` (右)
- 保持简洁，避免过宽表格

### 3.6 链接
- 相对链接（同一仓库内）:
  ```markdown
  [API参考](api/overview.md)
  ```
- 绝对链接（外部资源）:
  ```markdown
  [FastAPI官方文档](https://fastapi.tiangolo.com/)
  ```
- 锚点链接（跳转到文档内章节）:
  ```markdown
  [环境变量详情](#环境变量)
  ```
  注意：使用小写、连字符的锚点（自动生成）

### 3.7 图片
- 使用相对路径存储到仓库：
  ```markdown
  ![架构图](../images/architecture.png)
  ```
- 图片放在 `images/` 或 `assets/` 子目录
- 添加替代文本（alt text）：
  ```markdown
  ![数据同步流程图](images/data-sync-flow.png)
  ```
- 不要使用外部图片链接（避免死链）
- 大图考虑使用 thumbnails 链接到原图

### 3.8 脚注
使用 Markdown 脚注语法：
```markdown
这是一个引用[^1]。

[^1]: 这里是脚注内容，可以包含链接。
```

---

## 4. 内容模板

### 4.1 通用文档模板
```markdown
# [文档标题]

> 一句话摘要（可选）

## 概述
文档的简要介绍，说明目的和受众。

## 前置条件
- 要求1
- 要求2

## [主要内容章节]
...

## 常见问题
### Q: 问题?
A: 答案。

## 相关链接
- [相关文档](link)
- [外部资源](link)

## 更新历史
- 2026-03-02: 创建
```

---

### 4.2 操作指南模板（如何做 X）
```markdown
# 如何：[具体操作]

## 概述
简短说明这个操作的目的和场景。

## 步骤
1. **第一步** - 详细说明
   ```bash
   # 命令示例
   docker-compose up -d
   ```
2. **第二步** - 详细说明
3. ...

## 验证
完成后如何确认成功：
- 期望输出
- 检查点

## 常见问题
### Q: 错误信息?
A: 解决方案。

## 下一步
- [学习Y](guide-y.md)
- [查看Z](reference-z.md)
```

---

### 4.3 API端点文档模板
```markdown
# [端点名]: [描述]

**路径**: `[GET|POST|...] /api/endpoint`
**认证**: [required|optional]
**权限**: [列出所需权限]

## 请求参数
### 路径参数
| 名称 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | int | 是 | 策略ID |

### 查询参数（仅GET）
| 名称 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `page` | int | 否 | 页码（默认1） |

### 请求体（JSON）
```json
{
  "name": "string",
  "code": "string"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | str | 是 | 策略名称 |
| `code` | str | 是 | 策略代码 |

## 响应
### 成功响应 (200 OK)
```json
{
  "id": 1,
  "name": "MyStrategy"
}
```

### 错误响应
| 状态码 | 说明 |
|--------|------|
| 400 | 请求参数错误 |
| 401 | 未认证 |
| 403 | 权限不足 |
| 404 | 资源不存在 |

## 示例
```bash
curl -X POST http://localhost:8000/api/strategies \\
  -H "Authorization: Bearer YOUR_TOKEN" \\
  -H "Content-Type: application/json" \\
  -d '{"name":"test","code":"..."}'
```

## 注意
- 第1点
- 第2点
```

---

### 4.4 故障排查模板
```markdown
# [问题场景] 故障排查

## 症状
描述用户看到的问题（错误信息、异常行为）。

## 可能原因
1. 原因A
2. 原因B

## 诊断步骤
按顺序执行以下检查：

### 1. [检查什么]
```bash
# 命令
systemctl status service
```
**期望**: 输出应显示xxx
**如果不符合**: 进行到步骤2

### 2. [下一步检查]
...

## 解决方案
| 原因 | 解决步骤 |
|------|----------|
| 原因A | 1. 执行命令a<br>2. 重启服务 |
| 原因B | 联系管理员 |

## 预防措施
如何避免再次发生。

## 相关资源
- 相关文档链接
- 监控仪表板
```

---

## 5. 术语和缩写词典

### 5.1 项目特定术语
定义 TradderMate 特有的术语，统一使用：

| 术语 | 定义 | 使用规范 |
|------|------|----------|
| 策略 (Strategy) | vn.py策略类的封装 | 全称首次，后可简称 |
| 回测 (Backtest) | 历史数据上的策略模拟 | 使用中文术语 |
| 参数优化 | 参数空间搜索 | 同义词: 参数调优 |
| Tushare | 数据源API | 保留品牌名 |

**建议**: 在 `REFERENCE/GLOSSARY.md` 中维护完整术语表，所有文档引用。

---

### 5.2 技术缩写
| 缩写 | 全称 | 说明 |
|------|------|------|
| API | Application Programming Interface | 首次出现写全称 |
| JWT | JSON Web Token | 无需解释 |
| RQ | Redis Queue | 首次出现写全称 |
| CI | Continuous Integration | 常用，无需解释 |
| UX/UI | User Experience / User Interface | 无需解释 |
| vn.py | （保持原名） | 不翻译 |

---

## 6. 文档质量检查清单

在提交任何文档前，请检查：

### ✅ 内容
- [ ] 标题准确反映内容
- [ ] 概述部分清晰说明文档目的和受众
- [ ] 所有术语有定义或链接到术语表
- [ ] 代码示例可复制粘贴并正常工作
- [ ] 命令在目标环境中可执行
- [ ] 配置文件示例完整且正确
- [ ] 所有链接有效（运行 `markdown-link-check`）
- [ ] 无拼写和语法错误（使用拼写检查工具）
- [ ] 无敏感信息（密码、密钥）

### ✅ 格式
- [ ] 使用一致的标题层级（从H1开始）
- [ ] 代码块指定语言
- [ ] 表格格式正确、对齐
- [ ] 图片有替代文本且路径正确
- [ ] 无过长的行（软换行在80-100字符）
- [ ] 使用列表而非密集段落

### ✅ 用户体验
- [ ] 文档有逻辑流（从上到下可循序渐进）
- [ ] 关键部分有警告或注意框（如⚠️危险）
- [ ] 复杂过程分步骤
- [ ] 提供验证步骤（"完成后应看到..."）
- [ ] 有"下一步"或"相关链接"引导
- [ ] 包含FAQ或故障排查（如适用）

---

## 7. 特殊元素使用规范

### 7.1 警告和注意
使用 **Markdown强调 + emoji** 而非特殊块语法（保持兼容性）：

```
⚠️ **警告**: 这将删除数据！

✅ **提示**: 建议在测试环境先验证.

💡 **最佳实践**: 使用Docker避免污染本地环境.

📝 **注意**: 此功能在v1.2后可用.
```

---

### 7.2 环境变量
统一格式：
```markdown
## 环境变量

| 变量名 | 默认值 | 必填 | 说明 |
|--------|--------|------|------|
| `DATABASE_URL` | - | 是 | MySQL连接字符串 |
| `REDIS_HOST` | `localhost` | 否 | Redis主机 |

> **示例**: `export DATABASE_URL="mysql://user:pass@host/db"`
```

---

### 7.3 配置文件
展示完整示例，标注可修改部分：
```yaml
# docker-compose.yml
version: '3.8'
services:
  api:
    image: tradermate-api:latest
    environment:
      - DATABASE_URL=${DATABASE_URL}  # 从.env读取
      - DEBUG=${DEBUG:-false}         # 默认为false
```

---

### 7.4 Shell命令
- 单条命令使用代码块：
  ```bash
  docker-compose up -d
  ```
- 多步骤使用有序列表：
  ```bash
  # 1. 拉取镜像
  docker pull tradermate/api:latest

  # 2. 启动服务
  docker-compose up -d
  ```
- 输出示例使用 `#` 注释：
  ```bash
  $ python -m uvicorn app.main:app
  # 输出: INFO: Started server process [12345]
  ```

---

## 8. 语言和语气

### 8.1 语气指南
- **直接**: 使用主动语态（"使用docker-compose启动"而非"docker-compose应被使用"）
- **友好**: 使用"你"或"您"（根据受众），避免"用户"这种第三人称
- **包容**: 避免假设特定技术背景，解释缩写
- **鼓励**: 错误信息提供解决方案，而非责备

### 8.2 中英文混用
- 代码、API、技术术语: 保留英文（`FastAPI`, `Docker`, `API endpoint`）
- 概念和功能: 使用中文（"策略管理", "数据同步"）
- 品牌名: 保持原样（`Tushare`, `vn.py`, `React`）
- 界面元素: 按实际界面（如果前端有中文界面则用中文）

**示例**:
```markdown
在 **策略管理** 页面，你可以创建新的策略（Strategy）。
点击 **"New Strategy"** 按钮，填写名称和Python代码。
```

---

## 9. 生命周期管理

### 9.1 文档创建流程
1. 创建新文档，选择合适模板
2. 草稿完成后，Writer审查
3. 更新 `DOCUMENTATION_GAPS.md` 状态
4. 提交PR，需要至少一人审查
5. 合并后，在相关文档中添加链接（导航）

### 9.2 文档更新流程
1. 当功能变更时，**先更新文档**（文档即代码）
2. 更新 `CHANGELOG.md` 相应版本
3. 提交时使用 Conventional Commits:
   ```
   docs: 更新回测API参数说明
   ```
4. 如果重大变更（breaking change），更新 `BREAKING_CHANGES.md`

### 9.3 文档废弃流程
1. 用 `~~删除线~~` 标记为废弃
2. 在文档顶部添加警告：
   ```markdown
   ⚠️ **已废弃**: 此文档已过时，使用 [新文档](new-doc.md) 代替。
   ```
3. 3个月后无引用，可删除文件（但保留Git历史）

---

## 10. 审查清单（Reviewer视角）

当你审查文档时：

- [ ] **准确性**: 所有技术细节正确吗？在真实环境中验证过吗？
- [ ] **完整性**: 是否覆盖了所有重要场景？有无明显缺口？
- [ ] **清晰度**: 能否理解？能否复现步骤？
- [ ] **时效性**: 代码示例是否与当前代码库匹配？
- [ ] **安全性**: 有无泄露敏感信息（密钥、内网IP）？
- [ ] **规范性**: 是否符合本文档标准（格式、模板）？
- [ ] **链接**: 所有内部/外部链接是否有效？
- [ ] **受众**: 是否考虑了目标读者的知识水平？

---

## 11. 工具链（建议设置）

### 11.1 本地开发
```bash
# 安装markdownlint
npm install -g markdownlint-cli

# 检查所有文档
markdownlint "docs/**/*.md"

# 安装链接检查器
npm install -g markdown-link-check

# 验证链接
markdown-link-check -q "docs/**/*.md"
```

### 11.2 CI集成（GitHub Actions示例）
```yaml
name: Docs Check
on: [push, pull_request]
jobs:
  markdown-lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm install -g markdownlint-cli
      - run: markdownlint "docs/**/*.md"
  link-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm install -g markdown-link-check
      - run: markdown-link-check -q "docs/**/*.md"
```

---

## 12. 例外和解释

本标准适用于：
- ✅ 所有新增文档
- ✅ 重大更新的现有文档
- ⚠️ 已有文档将逐步迁移至本标准

如有特殊情况无法遵循本标准，请：
1. 在PR中说明原因
2. 与Writer协商解决方案
3. 记录例外情况（防止标准被侵蚀）

---

**批准**: 本标准由Main agent和Writer agent共同批准生效。

**修订**: 任何agent可提议修订，由Main和Writer审核。

---

**附录A**: 参考风格指南
- [Google Developer Documentation Style Guide](https://developers.google.com/style)
- [Microsoft Writing Style Guide](https://learn.microsoft.com/en-us/style-guide/)
- [撰写技术文档的最佳实践](https://www.writethedocs.org/guide/)

**附录B**: Markdown渲染差异
- 本项目使用GitHub Flavored Markdown (GFM)
- 某些平台（如GitLab）略有不同，但基本兼容
- 不依赖非标准扩展（如MDX、Pandoc特定语法）
