---
name: fork-sync
description: |
  管理 CNife/opencode fork 与上游 anomalyco/opencode 的同步。
  当用户说「合并 upstream」「同步上游」「更新 fork」「拉取上游变更」「merge upstream」时使用。
  也包括构建 opencode 二进制并部署的操作。
---

# Fork Sync — CNife/opencode

本 skill 封装了管理 CNife/opencode fork 与上游 anomalyco/opencode 同步的完整工作流，以及构建部署流程。

## 仓库配置

```
origin    → https://github.com/CNife/opencode.git
upstream  → https://github.com/anomalyco/opencode.git
默认分支  → dev
```

## 工作流

### 第一阶段：状态分析

在提出任何方案之前，先收集完整的仓库状态：

```bash
# 1. 基础状态
git status
git remote -v
git branch -a

# 2. 分歧量化
# 本地 vs origin
git log --oneline origin/dev..HEAD | head -5   # 本地独有
git log --oneline HEAD..origin/dev | head -5    # origin 独有

# 本地 vs upstream
git log --oneline dev..upstream/dev --count     # upstream 领先
git log --oneline upstream/dev..dev --count     # 本地领先
```

### 第二阶段：差异分析

向用户展示清晰的分歧概览，帮助决策：

```bash
# 1. 查看 origin 独有的 commit（时间、作者、内容）
git log --format="%h %ai %an %s" HEAD..origin/dev

# 2. 查看文件层面的差异
git diff --stat origin/dev..dev --diff-filter=M | head -20   # 修改的文件
git diff --stat origin/dev..dev --diff-filter=A | head -10   # 新增的文件
git diff --stat origin/dev..dev --diff-filter=D | head -10   # 删除的文件

# 3. 按模块分组展示差异，帮助评估冲突范围
# 例如：http-recorder、llm、opencode/src、sdk、web/docs 等
```

### 第三阶段：策略

只用 merge，禁止 rebase。

原因：fork 的 dev 分支已推送至 origin，rebase 会改写历史，导致其他协作者（或未来的自己）拉取时发生混乱。merge 保留真实的分叉拓扑，安全可追溯。

### 第四阶段：执行合并

分步执行，每次先告知用户将要做什么：

```bash
# 1. 处理未暂存文件
# bun.lock 是自动生成的，应该用 bun install 重新生成而非手动合并
git checkout -- bun.lock  # 丢弃未暂存修改

# 2. fetch 所有 remote
git fetch origin
git fetch upstream

# 3. 先合并 origin/dev（把 fork 上已有的变更合并进来）
git merge origin/dev -m "Merge origin/dev into dev"

# 4. 再合并 upstream/dev（把上游最新变更合并进来）
git merge upstream/dev -m "Merge upstream/dev into dev"
```

#### 冲突处理原则

| 文件类型 | 处理方式 |
|---------|---------|
| `bun.lock` | 自动生成，取任意一边，最后跑 `bun install` 重新生成 |
| 文档（`zen.mdx` 等） | 根据两边修改意图判断，通常保留 local 的定制 |
| 源码冲突 | 逐文件分析，理解两边改动意图后再解决 |

### 第五阶段：构建部署

```bash
# 构建单平台二进制
cd packages/opencode
bun run script/build.ts --single

# 安装到 PATH
cp dist/opencode-linux-x64/bin/opencode ~/.local/bin/opencode
chmod +x ~/.local/bin/opencode

# 验证
opencode --version
```

构建过程较慢（需要编译 Web UI + 二进制），建议在 tmux 中运行。

## 注意事项

- **merge-base 检查**：在分析分歧前先找到 merge-base，确认分叉点
- **同名 commit 陷阱**：如果本地和 origin 有同名但不同 hash 的 commit，用 `git diff <hash1> <hash2>` 确认内容是否真的一致
- **bun.lock**：永远不要手动解决 bun.lock 冲突，最后用 `bun install` 重新生成
- **typecheck**：推送前确认 typecheck 通过（项目 hook 会自动触发 `bun turbo typecheck`）
