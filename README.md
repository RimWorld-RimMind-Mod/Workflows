# RimMind Shared Workflows

RimMind 项目共用的 GitHub Actions 可复用工作流仓库。

## 可用工作流

### 1. Release 工作流 (`release.yml`)

自动生成 AI 驱动的 Release Notes 并打包 Mod 发布到 GitHub Releases。

**功能：**
- 根据 tag 自动计算 git 对比区间
- 收集区间内的 Commits 和 PR 数据
- 调用 DeepSeek AI 生成中英双语 Release Notes
- 按 `About/About.xml` 中的 Mod 名称打包 zip
- 自动判断是否为预发布版本（alpha/beta/rc/preview）
- 发布到 GitHub Releases 并附带 zip 产物

**输入参数：**

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `tag-name` | string | 是 | - | 触发发布的 tag 名称（如 `v1.2.3`） |
| `deepseek-base-url` | string | 否 | `https://api.deepseek.com` | DeepSeek API 基础 URL |
| `deepseek-model` | string | 否 | `deepseek-chat` | DeepSeek 模型名称 |
| `exclude-patterns` | string | 否 | `''` | 额外的 rsync 排除模式（空格分隔） |

**密钥：**

| 密钥 | 必填 | 说明 |
|------|------|------|
| `DEEPSEEK_API_KEY` | 是 | DeepSeek API 密钥 |
| `DEEPSEEK_BASE_URL` | 否 | DeepSeek API 基础 URL（优先于 input） |
| `GITHUB_TOKEN` | 是 | GitHub Token（用于创建 Release） |

**调用示例：**

```yaml
name: Release

on:
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+*'
      - 'V[0-9]+.[0-9]+.[0-9]+*'

permissions:
  contents: write
  pull-requests: read

jobs:
  release:
    uses: RimWorld-RimMind-Mod/Workflows/.github/workflows/release.yml@v1
    with:
      tag-name: ${{ github.ref_name }}
    secrets: inherit
```

**带额外排除模式的调用示例：**

```yaml
jobs:
  release:
    uses: RimWorld-RimMind-Mod/Workflows/.github/workflows/release.yml@v1
    with:
      tag-name: ${{ github.ref_name }}
      exclude-patterns: 'autotest/ .vscode/'
    secrets: inherit
```

### 2. Test 工作流 (`test.yml`)

运行 .NET 单元测试和分阶段架构测试。

**功能：**
- 自动检测 `Tests/` 和 `ArchTests/` 目录
- 使用 matrix 策略并行运行各阶段架构测试
- 支持自定义 .NET 版本、测试目录和阶段列表

**输入参数：**

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `dotnet-version` | string | 否 | `10.0.x` | .NET SDK 版本 |
| `test-dir` | string | 否 | `Tests` | 单元测试项目目录 |
| `arch-test-dir` | string | 否 | `ArchTests` | 架构测试项目目录 |
| `arch-phases` | string | 否 | `A,B,C,D,E,Full` | 要运行的阶段列表（逗号分隔） |

**架构测试阶段说明：**

| 阶段 | --filter 参数 | 说明 |
|------|---------------|------|
| A | `Phase=A\|Phase=General` | 基础架构验证 |
| B | `Phase=A\|Phase=B\|Phase=General` | 包含 A 阶段 |
| C | `Phase=A\|Phase=B\|Phase=C\|Phase=General` | 包含 A+B 阶段 |
| D | `Phase=A\|Phase=B\|Phase=C\|Phase=D\|Phase=General` | 包含 A+B+C 阶段 |
| E | `Phase=A\|Phase=B\|Phase=C\|Phase=D\|Phase=E\|Phase=General` | 包含 A+B+C+D 阶段 |
| Full | 无 filter | 运行全部架构测试 |

**调用示例：**

```yaml
name: Unit Tests

on:
  push:
    branches: [main, master]
    paths:
      - 'Source/**'
      - 'Tests/**'
      - 'ArchTests/**'
  pull_request:
    branches: [main, master]
    paths:
      - 'Source/**'
      - 'Tests/**'
      - 'ArchTests/**'

jobs:
  test:
    uses: RimWorld-RimMind-Mod/Workflows/.github/workflows/test.yml@v1
    secrets: inherit
```

**仅运行部分阶段的调用示例：**

```yaml
jobs:
  test:
    uses: RimWorld-RimMind-Mod/Workflows/.github/workflows/test.yml@v1
    with:
      arch-phases: 'A,B,Full'
    secrets: inherit
```

## 目录结构

```
Workflows/
├── .github/
│   └── workflows/
│       ├── release.yml      # 可复用 Release 工作流
│       └── test.yml         # 可复用 Test 工作流
├── scripts/
│   └── analyze_release.py   # AI Release Notes 生成脚本
├── .gitignore
└── README.md
```

## 注意事项

- 可复用工作流通过 `on: workflow_call` 定义，只能被其他仓库的 workflow 调用
- Release 工作流会自动 checkout 调用方仓库和本仓库（用于获取 `analyze_release.py` 脚本）
- 打包时会排除 `.shared-workflows/` 目录，不会将其包含在发布 zip 中
- `DEEPSEEK_API_KEY` 必须在调用方仓库的 Secrets 中配置
- 使用 `secrets: inherit` 可以简化密钥传递，调用方仓库的所有 secrets 会自动传递给可复用工作流
- 建议使用 `@v1` 引用而非 `@main`，以避免工作流更新导致的意外行为
