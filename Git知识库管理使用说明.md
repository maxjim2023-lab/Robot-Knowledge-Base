# Git 知识库管理使用说明

这份文档用于说明如何用 Git 管理 Obsidian 知识库，目标是让笔记可以被安全追踪、跨设备同步、随时回退，并减少误删、冲突和覆盖风险。

## 1. 基本原则

- 每次开始写笔记前，先拉取远端最新内容。
- 每次完成一段有效修改后，及时提交。
- 每次结束工作前，把本地提交推送到远端。
- 不把临时文件、缓存文件、私密文件提交进仓库。
- 出现冲突时先阅读内容，再手动合并，不要盲目覆盖。

推荐的日常节奏是：

```bash
git pull
# 编辑 Obsidian 笔记
git status
git add .
git commit -m "更新机器人学习笔记"
git push
```

## 2. 安装与检查

### Windows 安装 Git

如果终端提示 `git` 不是可识别命令，需要先安装 Git。

1. 打开 Git 官网下载安装包：https://git-scm.com/
2. 安装时保持默认选项即可。
3. 安装完成后重新打开 PowerShell 或终端。
4. 执行检查命令：

```bash
git --version
```

如果能看到版本号，例如 `git version 2.x.x`，说明安装成功。

### 首次配置身份

Git 提交需要记录作者信息。首次使用时配置一次即可：

```bash
git config --global user.name "你的名字"
git config --global user.email "你的邮箱"
```

查看配置：

```bash
git config --global --list
```

## 3. 初始化知识库仓库

如果当前 Obsidian 知识库还不是 Git 仓库，在知识库根目录执行：

```bash
git init
```

然后创建 `.gitignore` 文件，排除不适合提交的内容。

推荐的 Obsidian `.gitignore`：

```gitignore
# Obsidian workspace state
.obsidian/workspace.json
.obsidian/workspace-mobile.json

# Obsidian cache or plugin temporary files
.obsidian/cache/
.trash/

# OS files
.DS_Store
Thumbs.db
desktop.ini

# Temporary Office files
~$*.docx
~$*.xlsx
~$*.pptx

# Optional: large exported files
*.tmp
*.bak
```

如果你希望同步 Obsidian 插件、主题、快捷键和配置，可以保留 `.obsidian` 目录中的配置文件；如果只想同步笔记内容，可以进一步忽略 `.obsidian/`。

## 4. 连接远端仓库

常见远端平台包括 GitHub、Gitee、GitLab。

创建远端仓库后，在本地绑定远端地址：

```bash
git remote add origin 远端仓库地址
```

例如：

```bash
git remote add origin https://github.com/your-name/your-vault.git
```

第一次推送：

```bash
git branch -M main
git add .
git commit -m "初始化知识库"
git push -u origin main
```

之后只需要：

```bash
git push
```

## 5. 日常使用流程

### 开始写笔记前

先同步远端，尤其是多设备使用时：

```bash
git pull
```

如果提示有冲突，先解决冲突，再继续编辑。

### 查看改动

查看哪些文件被修改：

```bash
git status
```

查看具体修改内容：

```bash
git diff
```

### 暂存改动

暂存全部改动：

```bash
git add .
```

只暂存某个文件：

```bash
git add "Robot-Knowledge-Base/00-总览与路线图/00-机器人学习总览.md"
```

### 提交改动

```bash
git commit -m "更新 ROS2 学习笔记"
```

提交信息建议写清楚这次改了什么，不建议长期使用 `update`、`fix`、`1` 这类难以回看的信息。

推荐格式：

```text
新增 四足机器人步态笔记
更新 ROS2 常用命令速查
整理 AMR 路径规划章节
修正 强化学习术语说明
```

### 推送到远端

```bash
git push
```

## 6. 多设备同步规则

如果在电脑 A 和电脑 B 都编辑同一个知识库，建议遵守：

1. 每台设备开始前都执行 `git pull`。
2. 每台设备结束前都执行 `git add .`、`git commit`、`git push`。
3. 不要在两台设备上同时长时间编辑同一个文件。
4. 大改之前先提交一次当前稳定状态。

推荐顺序：

```bash
git pull
# 编辑
git status
git add .
git commit -m "更新今日学习笔记"
git push
```

## 7. 冲突处理

当两台设备修改了同一个文件的同一位置，`git pull` 时可能出现冲突。

冲突文件里会出现类似内容：

```text
<<<<<<< HEAD
本地版本内容
=======
远端版本内容
>>>>>>> origin/main
```

处理步骤：

1. 打开冲突文件。
2. 对比 `HEAD` 和 `origin/main` 两部分。
3. 手动保留正确内容。
4. 删除 `<<<<<<<`、`=======`、`>>>>>>>` 标记。
5. 保存文件。
6. 重新提交。

命令：

```bash
git status
git add .
git commit -m "解决笔记同步冲突"
git push
```

## 8. 回退与恢复

### 查看提交历史

```bash
git log --oneline
```

### 查看某个文件历史

```bash
git log -- "文件路径"
```

例如：

```bash
git log -- "Robot-Knowledge-Base/02-ROS2与机器人软件框架/07-常用命令速查.md"
```

### 恢复某个文件到上一次提交状态

如果你还没有提交，想放弃某个文件的本地修改：

```bash
git restore "文件路径"
```

### 恢复误删文件

如果文件被删除但还没提交：

```bash
git restore "文件路径"
```

### 回看旧版本内容

先查看提交记录：

```bash
git log --oneline
```

再查看某次提交中的文件：

```bash
git show 提交ID:"文件路径"
```

## 9. 分支使用

个人知识库通常只用 `main` 分支即可。

如果要进行大规模重构，例如调整目录结构、批量改名、拆分专题，可以新建分支：

```bash
git switch -c reorganize-notes
```

完成后提交：

```bash
git add .
git commit -m "重构机器人知识库目录"
```

切回主分支：

```bash
git switch main
```

合并：

```bash
git merge reorganize-notes
```

推送主分支：

```bash
git push
```

## 10. 常用命令速查

| 场景 | 命令 |
| --- | --- |
| 查看状态 | `git status` |
| 查看具体修改 | `git diff` |
| 暂存全部修改 | `git add .` |
| 提交修改 | `git commit -m "提交说明"` |
| 拉取远端更新 | `git pull` |
| 推送到远端 | `git push` |
| 查看提交历史 | `git log --oneline` |
| 恢复未提交文件 | `git restore "文件路径"` |
| 查看远端地址 | `git remote -v` |
| 新建分支 | `git switch -c 分支名` |
| 切换分支 | `git switch 分支名` |

## 11. 推荐工作习惯

### 小步提交

不要等一周后再提交一次。建议每完成一个相对完整的小主题就提交。

例如：

```bash
git commit -m "补充 MPC 与 WBC 对比"
git commit -m "新增 Unitree Go2 MuJoCo 笔记"
git commit -m "整理 ROS2 launch 文件章节"
```

### 提交前先检查

每次提交前执行：

```bash
git status
git diff
```

确认没有误提交临时文件、私密内容或无关文件。

### 大文件谨慎提交

Git 适合管理文本文件，不太适合频繁变动的大型二进制文件。

建议谨慎提交：

- 大型视频
- 大型压缩包
- 大型仿真数据
- 频繁更新的模型文件
- 自动生成的缓存文件

如果必须管理大文件，可以考虑 Git LFS。

## 12. Obsidian 使用注意事项

### 插件配置是否同步

`.obsidian` 目录包含 Obsidian 配置、插件、主题等内容。

推荐同步：

- `.obsidian/plugins/`
- `.obsidian/themes/`
- `.obsidian/hotkeys.json`
- `.obsidian/app.json`
- `.obsidian/appearance.json`

通常不推荐同步：

- `.obsidian/workspace.json`
- `.obsidian/workspace-mobile.json`

因为这些文件记录窗口布局、打开的标签页等状态，多设备之间容易互相覆盖。

### 附件目录

如果知识库中有图片、PDF、截图等附件，建议统一放在一个目录，例如：

```text
assets/
附件/
```

这样更容易管理和排查大文件。

### 文件命名

建议：

- 使用清晰标题。
- 避免同名文件。
- 少用特殊符号。
- 目录结构稳定后再大规模建立链接。

## 13. 推荐的一次完整操作示例

假设你今天更新了 ROS2 和四足机器人笔记：

```bash
git pull
git status
git diff
git add .
git commit -m "更新 ROS2 与四足机器人学习笔记"
git push
```

如果只是想确认有没有未提交内容：

```bash
git status
```

如果显示：

```text
nothing to commit, working tree clean
```

说明当前知识库已经没有未提交修改。

## 14. 故障排查

### git 命令不可用

现象：

```text
git : The term 'git' is not recognized
```

处理：

1. 安装 Git。
2. 重新打开终端。
3. 执行 `git --version`。
4. 如果仍然失败，检查 Git 是否加入系统 PATH。

### push 被拒绝

常见原因是远端有新内容，本地还没同步。

处理：

```bash
git pull
# 如有冲突，解决冲突
git push
```

### 不小心提交了不该提交的文件

如果只是想从 Git 追踪中移除，但保留本地文件：

```bash
git rm --cached "文件路径"
git commit -m "移除不应追踪的文件"
```

然后把它加入 `.gitignore`。

### 想知道最近改了什么

```bash
git log --oneline -5
```

查看最近一次提交详情：

```bash
git show
```

## 15. 最简记忆版

每天只需要记住这组命令：

```bash
git pull
git status
git add .
git commit -m "说明这次改了什么"
git push
```

一句话原则：

> 先拉取，再编辑；勤提交，常推送；遇冲突，先看内容再合并。
