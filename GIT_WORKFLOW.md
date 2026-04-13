# Git 工作流指南

本文档记录了本项目的 Git 分支管理和同步上游的工作流。

## 配置概览

### 远程仓库
- `origin`: https://github.com/VictorTQR/daily-paper-reader.git (你的 fork)
- `upstream`: https://github.com/ziwenhahaha/daily-paper-reader.git (上游原始仓库)

### 分支策略
- `main`: 保持与上游仓库同步，不进行任何自定义修改
- `dev`: 自定义开发分支，所有修改在此分支进行

---

## 日常开发流程

### 1. 开始开发
确保你在 dev 分支上：
```bash
git checkout dev
```

### 2. 提交修改
```bash
git add .
git commit -m "描述你的修改"
```

### 3. 推送到 origin
```bash
git push origin dev
# 或者首次推送时使用
git push -u origin dev
```

---

## 同步上游更新

当上游仓库有更新时，按以下步骤同步：

### 步骤 1: 更新 main 分支
```bash
# 切换到 main 分支
git checkout main

# 获取上游最新代码
git fetch upstream

# 将 main 分支 rebase 到 upstream/main
git rebase upstream/main

# 推送到你的 origin
git push origin main
```

### 步骤 2: 更新 dev 分支
```bash
# 切换到 dev 分支
git checkout dev

# 将 dev 分支 rebase 到最新的 main
git rebase main

# 解决冲突（如果有）后继续
git rebase --continue

# 强制推送到 origin（安全方式）
git push origin dev --force-with-lease
```

---

## 重要规则

1. **永远不在 main 分支上直接开发**
2. **所有自定义修改都在 dev 分支进行**
3. **同步时使用 rebase 保持历史整洁**
4. **推送 dev 分支时使用 --force-with-lease 而不是 --force**

---

## 首次设置（已完成）

本项目已按以下步骤配置：
```bash
# 1. 添加上游仓库
git remote add upstream https://github.com/ziwenhahaha/daily-paper-reader.git

# 2. 创建开发分支
git checkout -b dev
```
