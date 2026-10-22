# 第一课学习笔记

本课内容：Linux、Shell、Vim、Git 和 GitHub。

## 1. Linux 基础

Linux 是一个开源操作系统内核，Ubuntu 是常用的 Linux 发行版。控制组和视觉组后面会用到 ROS、GCC、CMake 等工具，所以需要熟悉 Linux 环境。

Linux 的目录从根目录 `/` 开始，常用的路径符号有：

| 符号 | 含义 |
| --- | --- |
| `/` | 根目录 |
| `~` | 当前用户的家目录 |
| `.` | 当前目录 |
| `..` | 上一级目录 |

Linux 的文件名区分大小写，例如 `TASK.md` 和 `task.md` 是两个不同的文件。以 `.` 开头的文件通常是隐藏文件，可以使用 `ls -a` 查看。

常用命令：

```bash
pwd                 # 查看当前路径
ls -lah             # 查看当前目录的文件
cd ~/projects       # 切换目录
mkdir -p demo/src   # 创建多级目录
touch test.txt      # 创建空文件
cp test.txt copy.txt
mv copy.txt new.txt
rm new.txt
```

需要特别注意，`rm` 删除的文件不会进入回收站，尤其是使用 `rm -r` 和 `sudo` 时要先确认路径。

### 文件权限

使用 `ls -l` 时，文件最前面会显示类似 `-rwxr-xr--` 的权限信息。三组权限分别对应文件所有者、同组用户和其他用户，`r`、`w`、`x` 分别表示读取、写入和执行。

```bash
ls -l system_report.sh
chmod +x system_report.sh       # 增加可执行权限
./system_report.sh              # 执行当前目录下的脚本
```

脚本即使内容正确，没有执行权限时也不能直接用 `./脚本名` 运行。不过仍然可以使用 `bash system_report.sh` 交给 Bash 执行。

遇到不熟悉的命令时，可以先查看帮助，而不是直接复制一条不了解的命令：

```bash
命令 --help
man 命令
```

## 2. Shell

终端是输入命令的窗口，Shell 负责解释和执行命令。Ubuntu 默认常用的是 Bash，Zsh 也是一种 Shell。

一个命令一般由命令名、选项和参数组成，例如：

```bash
ls -lah ~/projects
```

这里 `ls` 是命令，`-lah` 是选项，`~/projects` 是参数。

### 重定向和管道

```bash
date > output.txt           # 覆盖写入
echo "done" >> output.txt  # 追加写入
history | tail -n 10        # 将 history 的结果交给 tail
mkdir build && cd build     # 前一条成功后再执行后一条
```

`>` 和 `>>` 的区别需要记清楚：`>` 会覆盖原来的内容，`>>` 会在文件末尾追加。

其他比较实用的命令：

```bash
cat README.md
less README.md
head -n 10 README.md
tail -n 10 README.md
grep -i "error" robot.log
man ls
```

按 `Tab` 可以补全命令或路径，方向键上、下可以调用之前输入过的命令。

### 变量、引号和退出状态

Shell 变量在赋值时等号两边不能有空格，读取变量时在变量名前加 `$`：

```bash
name="DynamicX"
echo "$name"
```

双引号会保留变量替换，单引号会把其中的内容原样输出。路径变量通常要加双引号，避免路径中含有空格时被拆成多个参数。

每条命令结束后都会产生退出状态，`0` 表示成功，非 `0` 通常表示失败。可以用 `$?` 查看上一条命令的退出状态。`&&` 正是根据前一条命令是否成功来决定要不要继续执行。

### Shell 脚本的基本结构

```bash
#!/usr/bin/env bash

file="notes.md"

if [ -f "$file" ]; then
    echo "文件存在"
else
    echo "没有找到文件"
    exit 1
fi
```

第一行称为 Shebang，用来说明脚本应由哪个解释器执行。`if` 用于条件判断，`-f` 用来检查目标是否为普通文件。脚本出现无法继续处理的情况时，可以用 `exit 1` 主动返回失败状态。

## 3. Vim

Vim 是终端文本编辑器。刚开始最容易出错的地方是没有分清当前模式。

- 普通模式：打开 Vim 后默认进入，可以移动光标、复制和删除。
- 插入模式：按 `i`、`a` 或 `o` 进入，用来输入文字。
- 命令行模式：先按 `Esc` 回到普通模式，再按 `:` 进入。

常用按键：

```text
h j k l     左、下、上、右移动
dd          删除当前行
yy          复制当前行
p           粘贴
u           撤销
gg / G      跳到文件开头/结尾
/word       搜索 word
```

保存和退出：

```text
:w      保存
:q      退出
:wq     保存并退出
:q!     不保存，强制退出
```

如果不知道自己在哪个模式，可以先按几次 `Esc`，再输入需要的命令。

基本编辑顺序：打开文件 → 按 `i` 输入 → 按 `Esc` 回到普通模式 → 输入 `:w` 保存 → 检查文件 → 输入 `:q` 退出。

## 4. Git 和 GitHub

Git 用来记录代码的修改历史，GitHub 用来托管远程仓库和完成多人协作。两者的关系可以简单理解为：Git 负责本地版本管理，GitHub 提供远程仓库和 Pull Request 等功能。

一次提交会经过下面几个位置：

```text
工作区 --git add--> 暂存区 --git commit--> 本地仓库 --git push--> 远程仓库
```

常用命令：

```bash
git clone 仓库地址
git status
git diff
git add 文件名
git commit -m "Add first lesson notes."
git log --oneline
git push
```

`git status` 很重要，可以查看当前分支、修改过的文件和还没有提交的文件。`git add` 之后还可以使用 `git diff --cached` 检查即将提交的内容。

### Fork 和 Pull Request

这次作业使用的流程是：

1. Fork 原仓库到自己的 GitHub 账号；
2. Clone 自己的仓库到本地；
3. 创建新的开发分支；
4. 完成笔记和脚本；
5. 使用 `git add`、`git commit` 和 `git push` 提交；
6. 在 GitHub 上向原仓库发起 Pull Request。

本次使用的分支命令：

```bash
git switch -c feature/luo070526-sudo-first-lesson
git add 2027/luo070526-sudo
git diff --cached
git commit -m "Complete first lesson homework."
git push -u origin feature/luo070526-sudo-first-lesson
```

如果 PR 提交后又发现问题，只需要继续在原分支修改并 Push，原来的 PR 会自动更新，不用关闭后重新创建。

同步原仓库前，先添加 `upstream`：

```bash
git remote add upstream https://github.com/OceanCing/missing-semester.git
git fetch upstream
```

同步上游仓库时，可以先取回更新，再把自己的分支建立在最新代码上。操作前应使用 `git status` 确保重要修改已经提交或暂存，避免在工作区混乱时处理分支。

```bash
git fetch upstream
git switch master
git merge upstream/master
```

### 查看修改和处理冲突

提交前依次检查：

```bash
git status             # 哪些文件发生了变化
git diff               # 还没有加入暂存区的修改
git diff --cached      # 即将进入下一次提交的修改
git log --oneline -5   # 最近的提交记录
```

当两条分支修改了同一个文件的同一位置时，Git 可能无法自动决定保留哪一边，这时就会产生冲突。发生冲突后，文件中会出现 `<<<<<<<`、`=======` 和 `>>>>>>>` 标记。需要根据实际内容手动整理，删掉冲突标记，再执行 `git add` 和 `git commit`。

处理冲突时不能只为了让 Git 不报错就随便删除一边，应该先看清楚两边分别改了什么，并在编译或运行检查通过后再提交。

## 5. 本次作业

目录结构：

```text
2027/luo070526-sudo/
├── notes/
│   ├── first_lesson_notes.md
│   └── cpp_basics_notes.md
└── homework/
    ├── system_report.sh
    └── output.txt
```

`system_report.sh` 会输出运行时间、最近使用的 10 条命令和笔记字数，并把结果重定向到同一目录下的 `output.txt`。

脚本通过 `dirname "$0"` 获取自身所在目录，再定位笔记和输出文件，避免受当前工作目录影响。脚本会检查笔记目录，并兼容 Bash 和 Zsh 的历史记录文件。

本次检查步骤：

```bash
bash -n 2027/luo070526-sudo/homework/system_report.sh
chmod +x 2027/luo070526-sudo/homework/system_report.sh
./2027/luo070526-sudo/homework/system_report.sh
git diff --check
```

`bash -n` 只检查语法而不执行脚本；`git diff --check` 可以发现多余的行尾空格等格式问题。这些检查虽然简单，但能在提交前排除一部分常见错误。

## 参考资料

- [培训视频：27 赛季控制视觉招新培训第一课](https://www.bilibili.com/video/BV1Uwn36PE7c/)
- [作业仓库：OceanCing/missing-semester](https://github.com/OceanCing/missing-semester)
- [Pro Git](https://git-scm.com/book/zh/v2)
