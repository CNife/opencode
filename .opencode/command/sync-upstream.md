---
description: 拉取 upstream 最新变更，合并最新 release tag 到 dev 分支，推送到远程
---

拉取 upstream 最新变更，合并最新的 release tag 到当前 dev 分支，然后推送到 origin。

流程：

1. 确认当前在 `dev` 分支且工作区干净（`git status`）。如果不是，先 stash 或报错。
2. `git fetch upstream --tags --prune` 拉取 upstream 所有变更和 tags。
3. 找到最新的 `v1.x.x` 格式 tag：`git tag -l 'v1.*' --sort=-v:refname | head -1`。
4. `git merge <tag> --no-edit` 合并到 dev。
5. 如果 `bun.lock` 有冲突，不要手动选 theirs/ours，而是：
   - `git checkout --theirs bun.lock` 暂解冲突
   - `bun install` 让 bun 重新解析依赖生成正确的 lockfile
   - `git add bun.lock`
6. `git commit --no-edit` 完成 merge。
7. `git push origin dev` 推送到远程。

注意事项：
- 所有 git 命令使用环境变量确保非交互模式：`CI=true GIT_TERMINAL_PROMPT=0 GCM_INTERACTIVE=never HOMEBREW_NO_AUTO_UPDATE=1 GIT_EDITOR=: EDITOR=: VISUAL='' GIT_SEQUENCE_EDITOR=: GIT_MERGE_AUTOEDIT=no GIT_PAGER=cat PAGER=cat`
- 如果 merge 有其他冲突（非 bun.lock），报告给用户，不要自行解决。
- 推送成功后简要汇报：合并了哪个 tag，有多少新提交。
