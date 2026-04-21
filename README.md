# pr-workflows

基于 Claude AI 的 GitHub Actions 可复用工作流库，为团队所有项目提供统一的 PR 自动化审查与 CI/CD 能力。

## 包含的工作流

| 工作流 | 触发条件 | 功能 |
|--------|---------|------|
| `claude-review.yml` | PR 创建、代码更新、标记 Ready（跳过 Draft） | 全量代码审查，覆盖代码质量、安全性、性能、测试 |
| `claude-security.yml` | PR 涉及敏感路径变更 | 深度安全审查，对照 OWASP Top 10 逐项检查 |
| `claude-ontology-review.yml` | PR 创建、代码更新、标记 Ready（仅 Java 项目） | 本体设计与代码实现一致性审核，审核完成后云之家通知 PR 作者 |
| `claude.yml` | Issue 或 PR 评论中包含 `@claude` | AI 实时交互，支持代码解释、方案讨论等 |
| `jenkins-deploy.yml` | push to test 分支 | 触发 Jenkins 部署构建并等待结果 |
| `jenkins-autotest.yml` | 通常由 jenkins-deploy.yml 串联触发 | 触发 Jenkins 自动化测试并等待结果 |

审查结果以中文输出，具体问题通过 inline comment 标注到代码行，整体评价通过 PR comment 汇总。

## 代码规范

工作流在审查时会自动加载代码规范文档注入给 AI，规范来源优先级：

1. **子项目自有规范**：子项目 `.github/standards/*.md`（存在则使用，不再加载共享规范）
2. **共享默认规范**：本仓库 `standards/*.md`（子项目无自有规范时使用）

如需为子项目定制规范，在其 `.github/standards/` 目录下放置 `.md` 文件即可，支持多个文件。

## 接入方式

Secrets 已在组织层面统一配置，子项目无需重复设置。如需使用独立的 API 密钥，在子项目仓库的 **Settings → Secrets and variables → Actions** 中覆盖以下变量即可：

| 密钥 | 说明 |
|------|------|
| `ANTHROPIC_API_KEY` | Anthropic API 访问密钥 |
| `ANTHROPIC_BASE_URL` | 自定义 API 端点（使用代理时填写） |
| `YUNZHIJIA_NOTIFY_URL` | 云之家通知接口地址 |
| `YUNZHIJIA_ACCESS_TOKEN` | 云之家通知 Token |
| `JENKINS_URL` | Jenkins 代理地址 |
| `JENKINS_USER` | Jenkins 用户名 |
| `JENKINS_API_TOKEN` | Jenkins API Token |
| `JENKINS_PIPELINE_TOKEN` | 流水线触发 Token |

### 创建工作流文件

在子项目 `.github/workflows/` 下按需创建以下文件：

**claude-review.yml** — PR 自动代码审查

```yaml
name: Claude Auto Review

on:
  pull_request:
    types: [opened, synchronize, ready_for_review]

jobs:
  review:
    uses: invagent/pr-workflows/.github/workflows/claude-review.yml@master
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      ANTHROPIC_BASE_URL: ${{ secrets.ANTHROPIC_BASE_URL }}
```

**claude-security.yml** — 安全敏感路径审查（按实际路径修改 `paths`）

```yaml
name: Claude Security Review

on:
  pull_request:
    types: [opened, synchronize]
    paths:
      - 'src/auth/**'
      - 'src/api/**'
      - 'src/payment/**'
      - 'src/middleware/**'
      - 'config/**'

jobs:
  security:
    uses: invagent/pr-workflows/.github/workflows/claude-security.yml@master
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      ANTHROPIC_BASE_URL: ${{ secrets.ANTHROPIC_BASE_URL }}
```

**claude-ontology-review.yml** — 本体一致性审核（仅 Java 项目）

```yaml
name: Claude Ontology Review

on:
  pull_request:
    types: [opened, synchronize, ready_for_review]
    branches:
      - test  # 只对目标分支为 test 的 PR 触发

jobs:
  ontology-review:
    permissions:
      contents: read
      pull-requests: write
      id-token: write
    uses: invagent/pr-workflows/.github/workflows/claude-ontology-review.yml@master
    with:
      # 可选：指定本体对象 slug，留空则自动探测
      ontology_objects: ''
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      ANTHROPIC_BASE_URL: ${{ secrets.ANTHROPIC_BASE_URL }}
      YUNZHIJIA_NOTIFY_URL: ${{ secrets.YUNZHIJIA_NOTIFY_URL }}
      YUNZHIJIA_ACCESS_TOKEN: ${{ secrets.YUNZHIJIA_ACCESS_TOKEN }}
```

**claude.yml** — `@claude` 交互

```yaml
name: Claude Code

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  pull_request_review:
    types: [submitted]
  issues:
    types: [opened]

jobs:
  claude:
    uses: invagent/pr-workflows/.github/workflows/claude.yml@master
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      ANTHROPIC_BASE_URL: ${{ secrets.ANTHROPIC_BASE_URL }}
```

**jenkins-deploy.yml** — 部署构建，可选串联自动化测试；构建失败时云之家通知触发人

```yaml
name: Trigger Jenkins Deploy

on:
  push:
    branches:
      - test

jobs:
  deploy:
    uses: invagent/pr-workflows/.github/workflows/jenkins-deploy.yml@master
    with:
      service: 'your-service-name'
      branch: 'test'
      deploy: 'true'
    secrets:
      JENKINS_URL: ${{ secrets.JENKINS_URL }}
      JENKINS_USER: ${{ secrets.JENKINS_USER }}
      JENKINS_API_TOKEN: ${{ secrets.JENKINS_API_TOKEN }}
      JENKINS_PIPELINE_TOKEN: ${{ secrets.JENKINS_PIPELINE_TOKEN }}
      YUNZHIJIA_NOTIFY_URL: ${{ secrets.YUNZHIJIA_NOTIFY_URL }}
      YUNZHIJIA_ACCESS_TOKEN: ${{ secrets.YUNZHIJIA_ACCESS_TOKEN }}

  # 可选：部署成功后触发自动化测试（去掉此 job 则不触发）
  autotest:
    needs: deploy
    if: success()
    uses: invagent/pr-workflows/.github/workflows/jenkins-autotest.yml@master
    with:
      run_mode: 'issue_invoice'
      issue_mode: 'smoke'
      issue_type: 'blue'
    secrets:
      JENKINS_URL: ${{ secrets.JENKINS_URL }}
      JENKINS_USER: ${{ secrets.JENKINS_USER }}
      JENKINS_API_TOKEN: ${{ secrets.JENKINS_API_TOKEN }}
      JENKINS_PIPELINE_TOKEN: ${{ secrets.JENKINS_PIPELINE_TOKEN }}
      YUNZHIJIA_NOTIFY_URL: ${{ secrets.YUNZHIJIA_NOTIFY_URL }}
      YUNZHIJIA_ACCESS_TOKEN: ${{ secrets.YUNZHIJIA_ACCESS_TOKEN }}
```

所有工作流按需选用，不必全部接入。
