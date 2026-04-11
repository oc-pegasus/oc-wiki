# Fork 仓库工作流：main / deploy / feat 分支策略

## 场景

你 fork 了别人的项目，需要：
- 跟上游保持同步
- 维护自己的定制改动（配置、额外功能）
- 偶尔给上游提 PR

## 分支策略

```
upstream/main ──→ origin/main (镜像同步，不直接提交)
                     │
                     ├──→ origin/deploy (main + 定制配置 + feature)
                     │      生产部署用这个分支
                     │
                     └──→ origin/feat/xxx (从 upstream/main 创建)
                            给上游提 PR 用
```

### main 分支
- **只做一件事**：跟 upstream/main 保持完全一致
- 不要直接在 main 上提交任何东西
- 同步方式：`git fetch upstream && git merge upstream/main`

### deploy 分支
- 你的生产分支，包含所有定制内容
- 基于 main，加上你的配置和功能改动
- upstream 更新时：先同步 main，再 merge main 到 deploy

### feat/xxx 分支
- **必须从 upstream/main（或你的 main）创建**，不要从 deploy 创建
- 否则 PR 会带上所有 deploy 分支的 commits

## 踩坑

### PR 带了不相关的 commits
**原因**：从 deploy 分支创建了 feature 分支
**正确做法**：
```bash
git fetch upstream
git checkout -b feat/my-feature upstream/main
# 在这里开发
git push origin feat/my-feature
# 然后从 feat/my-feature 向 upstream/main 提 PR
```

### upstream 更新后 deploy 分支冲突
```bash
git checkout main
git fetch upstream
git merge upstream/main
git push origin main

git checkout deploy
git merge main
# 解决冲突
git push origin deploy
```

## 要点
- main = 镜像，deploy = 生产，feat = PR
- feat 分支永远从 main/upstream 创建
- 职责分离，不要混用
