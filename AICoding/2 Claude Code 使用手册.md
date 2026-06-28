# 使用技巧

## 做任何事之前先运行 /init
每个新项目里的第一件事：运行 /init。
Claude 会读取你的代码库，并生成一个 CLAUDE.md 文件。

这个文件会成为项目大脑，记录架构决策、约定、模式和命名规则。
之后的每一次会话，都会从已经加载好的上下文开始。

## 用 /statusline 看清当前状态
配置 /statusline，把终端变成实时仪表盘。
你可以实时看到 token 使用量、会话指标、模型信息。

## 启用语音输入
免手动编码能让你一直停留在解决问题的状态。不用停下来打字，你可以直接说出下一条指令。
在设置里启用一次，之后持续使用。

## 用 /context 找出是谁在消耗 token
你的上下文窗口是一份预算。
当它变重时，用 /context 检查到底是什么在占用它。

##  60% 时压缩，不同任务之间清空
上下文管理有两条规则：
→ 当上下文达到 60% 时运行 /compact：Claude 会压缩对话，同时保留关键信息→ 在完全无关的任务之间运行 /clear：重新开始

## 复杂任务从计划模式开始
在 Claude 写下第一行代码之前，先让它制定计划。
```
# 开启计划模式
/plan

# 关闭/返回编辑模式
/edit
```

为什么有效：
→ 迫使 Claude 先思考再行动→ 提前暴露不一致的假设→ 在消耗计算资源之前，给你一个校准点
以计划模式开始的会话，会多花 3 分钟。
不做计划就开始的会话，往往要多花 3 小时补救。


## 开始之前，让 Claude 先提问
把这段加到每一个复杂提示词里：
```
开始之前，请先提出你需要我澄清的问题，
这样你才能把这件事做好。请一次性列出所有问题。
```

## 把自检写进待办清单
给 Claude 任务清单时，加入验证步骤：
```
Todo:
1. 构建支付表单
2. 添加输入校验
3. ✓ 验证：用空输入、特殊字符和超过 $10,000 的金额测试
4. 添加错误提示
5. ✓ 验证：确认所有错误状态都已处理并且可见

```

## 部署子智能体，并行处理任务
这是最容易被低估的功能之一。
与其让任务按顺序执行，不如把它们拆给多个拥有隔离上下文的子智能体。
```
"启动一个子智能体处理数据库结构。
再启动一个处理 API 端点。
你来协调集成。"
```

## 把可复用提示词保存成自定义技能
同一套指令只要写第二遍，你就在浪费时间。
把可复用工作流保存成技能文件：
```
# skills/code-review.md
审查代码时：
1. 先检查安全漏洞
2. 标出所有 N+1 查询
3. 验证每个外部调用的错误处理
4. 检查变量命名是否符合我们的约定（camelCase、描述性命名）
5. 输出格式：先列严重问题，再给建议
```

##  在 CLAUDE.md 中链接外部文件，不要全部贴进去
不要把所有内容都塞进 CLAUDE.md。
引用外部文件：
```
# CLAUDE.md

## 架构
参见：/docs/architecture.md

## API 约定
参见：/docs/api-standards.md

## 当前迭代
参见：/docs/sprint-23.md
```

Claude 只会加载需要的部分。

## 挑战第一版输出
不要直接接受结果。
每次输出之后都问：
```
这份结果最薄弱的部分是什么？
如果你有更多时间，会改哪里？
有没有更简单的方案值得考虑？
```

## 集成 Chrome DevTools
让 Claude 通过 DevTools 与正在运行的应用交互。
它可以实时读取控制台错误、网络请求和 DOM 状态。
```
"在结账页打开 DevTools。
找出支付按钮没有响应的原因。
检查控制台错误和 Network 面板。"
```

## 恢复之前的会话
不要丢掉上一次会话里的上下文。
```
claude --resume 9901b366-c4b6-4d78-89ad-81e964e373e
```

## 使用 Git Worktrees 运行并行会话

对复杂项目来说，这是最大的生产力解锁点。
在不同分支上同时运行多个 Claude 会话。
```
# 为并行工作创建 worktree
git worktree add ../project-feature-auth feature/auth
git worktree add ../project-feature-payments feature/payments
git worktree add ../project-feature-dashboard feature/dashboard

# 然后在每个目录里运行独立的 Claude 会话
# Claude 会同时处理三条线
# 没有冲突。不用等待。
```

### Git Worktrees
Git Worktree 让你在同一个仓库下同时检出多个分支，每个分支有自己独立的工作目录。
和 git clone 多份的区别：
- git clone 多份：每份都有完整的 .git 目录，占用大量磁盘空间，分支之间互不知道
- git worktree：共享同一个 .git 目录，几乎不占额外空间，分支之间可以互相看到

目录结构示例：
```
my-project/              # 主工作区，你自己在这里开发
../worktrees/
  ├── feature-auth/      # Claude 1 在这里做认证功能
  ├── fix-bug-123/       # Claude 2 在这里修 bug
  └── refactor-api/      # Claude 3 在这里重构 API
```
每个目录都是完整的工作区，可以独立运行、独立提交、独立启动 Claude Code。

形象点说：就像你有一个大办公室（.git 目录），里面有很多小隔间（worktree），每个隔间都在干不同的活，但共享同一套基础设施（Git 历史、配置等）。

### 基本用法
创建 worktree:
```
# 创建新分支并检出到独立目录
git worktree add ../worktrees/feature-auth -b feature/auth

# 基于已有分支创建
git worktree add ../worktrees/fix-bug origin/fix-bug-123

# 基于某个 commit 创建
git worktree add ../worktrees/hotfix abc123
```

在 worktree 中启动 Claude Code:
```
cd ../worktrees/feature-auth
claude
```

这样就有了一个独立的 Claude Code 实例，在 feature/auth 分支上工作。你可以开多个终端，每个终端进入不同的 worktree，启动不同的 Claude Code 实例。

查看和清理：
```
# 查看所有 worktree
git worktree list

# 删除 worktree（会保留分支）
git worktree remove ../worktrees/feature-auth

# 清理失效的 worktree 引用
git worktree prune
```

合并代码：
```
# 1 通过 PR（推荐）
cd ../worktrees/feature-auth
git add .
git commit -m "feat: implement user authentication"
git push -u origin feature/auth

# 创建 PR（需要安装 GitHub CLI）
gh pr create --title "Add user authentication" --body "实现用户认证功能"


# 2 直接 merge
cd ~/projects/my-project  # 回到主工作区
git checkout main
git merge feature/auth
git push
```


