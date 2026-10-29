---
tags:
  - type/lesson
  - topic/linux
  - topic/git
  - status/done
date: 2026-10-29
---

# 第一课：Linux、Shell、Vim 与 Git

这节课的主线不是背命令，而是掌握一套开发工作流：

```text
在 Linux 中定位文件
  → 用 Shell 执行操作
  → 用 Vim 修改文本
  → 用 Git 记录版本
  → 通过 GitHub Pull Request 协作
```

## 1. Linux 目录与常用命令

Linux 的所有文件都从根目录 `/` 开始。Ubuntu 是 Linux 的一种发行版，ROS、GCC、CMake 等控制组工具主要运行在 Linux 环境中。

| 符号 | 含义 |
| --- | --- |
| `/` | 根目录 |
| `~` | 当前用户的家目录 |
| `.` | 当前目录 |
| `..` | 上一级目录 |

`cd` 是 **change directory** 的缩写，用来切换当前目录：

```bash
pwd               # 显示当前所在目录
ls -lah           # 查看当前目录内容
cd ~/下载         # 进入家目录下的“下载”
cd ..             # 返回上一级目录
cd -              # 返回上一次所在目录
cd                # 不带参数时回到家目录
```

路径必须与真实名称一致。我的 Ubuntu 使用中文目录 `~/下载`，所以 `cd ~/Downloads` 会提示目录不存在。

常用文件操作：

```bash
mkdir -p demo/src
touch demo/src/main.cpp
cp demo/src/main.cpp demo/src/main_copy.cpp
mv demo/src/main_copy.cpp demo/src/test.cpp
rm demo/src/test.cpp
```

`rm` 删除的文件通常不会进入回收站。执行 `rm -r` 或带 `sudo` 的删除命令前，先用 `pwd` 和 `ls` 确认目标。

### 1.1 文件权限

`ls -l` 显示的 `r`、`w`、`x` 分别表示读取、写入和执行权限。

```bash
ls -l system_report.sh
chmod +x system_report.sh
./system_report.sh
```

没有执行权限时，也可以明确交给 Bash：

```bash
bash system_report.sh
```

## 2. Shell

终端是输入命令的窗口，Shell 负责解释命令。Ubuntu 常用 Bash。

一条命令通常由“命令名＋选项＋参数”组成：

```bash
ls -lah ~/projects
```

- `ls`：命令；
- `-lah`：选项；
- `~/projects`：参数。

### 2.1 重定向、管道与条件执行

```bash
date > output.txt
echo "done" >> output.txt
history | tail -n 10
mkdir build && cd build
```

- `>`：覆盖写入文件；
- `>>`：追加到文件末尾；
- `|`：把前一条命令的输出交给后一条命令；
- `&&`：前一条成功后才执行后一条。

每条命令都有退出状态。`0` 表示成功，非 `0` 通常表示失败：

```bash
echo $?
```

### 2.2 Shell 脚本典型结构

```bash
#!/usr/bin/env bash

note_file="notes/first_lesson_notes.md"

if [ -f "$note_file" ]; then
    wc -w "$note_file"
else
    echo "没有找到笔记"
    exit 1
fi
```

- Shebang 指定解释器；
- `[ -f ... ]` 检查普通文件是否存在；
- 路径变量加双引号，避免空格把路径拆开；
- `exit 1` 主动返回失败状态。

## 3. Vim

Vim 最重要的是分清模式：

| 模式 | 作用 | 进入方法 |
| --- | --- | --- |
| 普通模式 | 移动、复制、删除 | 打开文件后默认进入，或按 `Esc` |
| 插入模式 | 输入文字 | 按 `i`、`a` 或 `o` |
| 命令行模式 | 保存、退出、搜索 | 普通模式下按 `:` 或 `/` |

常用操作：

```text
i       进入插入模式
Esc     回到普通模式
dd      删除当前行
yy      复制当前行
p       粘贴
u       撤销
/word   搜索 word
:w      保存
:q      退出
:wq     保存并退出
:q!     放弃修改并退出
```

如果不知道当前在哪个模式，先按 `Esc`，再输入保存或退出命令。

## 4. Git 与 GitHub

Git 负责本地版本管理，GitHub 提供远程仓库、代码审查和 Pull Request。

```text
工作区 --git add--> 暂存区 --git commit--> 本地仓库 --git push--> 远程仓库
```

提交前的检查顺序：

```bash
git status
git diff
git add 2027/luo070526-sudo
git diff --cached
git commit -m "Complete first lesson homework."
git push
```

- `git status`：查看分支和文件状态；
- `git diff`：查看尚未暂存的修改；
- `git diff --cached`：查看即将提交的内容；
- `git commit`：保存一次有意义的版本；
- `git push`：把本地提交发送到远程分支。

### 4.1 本次 Fork 与 PR 工作流

```bash
git switch -c feature/luo070526-sudo-first-lesson
git push -u origin feature/luo070526-sudo-first-lesson
```

完整流程是：

1. Fork 原仓库；
2. Clone 自己的 Fork；
3. 在开发分支修改；
4. Commit 并 Push；
5. 向原仓库发起 Pull Request；
6. 后续继续 Push 到原分支，原 PR 会自动更新。

PR 已经建立后不应为了更新作业而关闭重开。

### 4.2 同步上游与处理冲突

```bash
git remote add upstream https://github.com/OceanCing/missing-semester.git
git fetch upstream
git switch master
git merge upstream/master
```

当两个分支修改同一文件的同一位置时，Git 可能留下：

```text
<<<<<<<
自己的内容
=======
上游内容
>>>>>>>
```

需要判断两边内容，手动合并并删除标记，再执行 `git add` 和 `git commit`。不能为了消除报错随便删除一边。

## 5. 第一课作业

```text
2027/luo070526-sudo/
├── notes/
│   ├── first_lesson_notes.md
│   └── cpp_basics_notes.md
└── homework/
    ├── system_report.sh
    └── output.txt
```

`system_report.sh` 需要把运行时间、最近 10 条命令和笔记字数重定向到同目录的 `output.txt`。

提交前验证：

```bash
bash -n 2027/luo070526-sudo/homework/system_report.sh
chmod +x 2027/luo070526-sudo/homework/system_report.sh
./2027/luo070526-sudo/homework/system_report.sh
git diff --check
```

- `bash -n`：只检查脚本语法；
- `git diff --check`：检查多余的行尾空格等格式问题。

## 6. 本课形成的习惯

- 不确定路径时先用 `pwd`、`ls` 和 `find`；
- 不复制自己无法解释的高风险命令；
- 修改后先看 `git diff`，提交前再看 `git diff --cached`；
- 一次 Commit 只表达一件清楚的事情；
- PR 收到意见后继续修改原分支，不关闭重开。

## 参考资料

- [27 赛季控制视觉招新培训第一课](https://www.bilibili.com/video/BV1Uwn36PE7c/)
- [OceanCing/missing-semester](https://github.com/OceanCing/missing-semester)
- [Pro Git](https://git-scm.com/book/zh/v2)
