# 🐧 Linux 常用终端操作指令

> 一份随时查阅的终端命令速查表，涵盖文件操作、文本处理、Git 等常用场景。

---

## 📁 一、文件与目录操作

### 基本导航

| 命令 | 说明 |
|:-----|:-----|
| `pwd` | 显示当前工作目录的绝对路径 |
| `ls` | 列出当前目录下的文件和文件夹 |
| `ls -la` | 列出所有文件（含隐藏文件），详细模式 |
| `ls -lh` | 以人类可读的大小格式（KB / MB / GB）列出 |
| `ls -lt` | 按修改时间排序，最新的在最前 |
| `cd <dir>` | 切换到指定目录 |
| `cd ..` | 返回上一级目录 |
| `cd ~` | 返回当前用户的家目录 |
| `cd -` | 返回上一次所在的目录 |

### 文件查看

| 命令 | 说明 |
|:-----|:-----|
| `cat <file>` | 一次性显示整个文件内容 |
| `less <file>` | 分页查看文件，支持上下滚动（`q` 退出） |
| `head -n <N> <file>` | 查看文件前 N 行 |
| `tail -n <N> <file>` | 查看文件后 N 行 |
| `tail -f <file>` | 实时追踪文件末尾新增内容（常用于看日志） |
| `wc -l <file>` | 统计文件行数 |
| `wc -w <file>` | 统计文件词数 |

### 文件创建与删除

| 命令 | 说明 |
|:-----|:-----|
| `touch <file>` | 创建一个空文件，或更新文件的时间戳 |
| `mkdir <dir>` | 创建一个目录 |
| `mkdir -p <a/b/c>` | 递归创建多级目录 |
| `rm <file>` | 删除文件 |
| `rm -r <dir>` | 递归删除目录及其内容 |
| `rm -rf <dir>` | ⚠️ 强制递归删除，不提示确认 — **非常危险** |
| `rmdir <dir>` | 删除空目录（目录非空时会报错，比 `rm -r` 安全） |

### 文件复制与移动

| 命令 | 说明 |
|:-----|:-----|
| `cp <src> <dst>` | 复制文件 |
| `cp -r <src> <dst>` | 递归复制整个目录 |
| `cp -i <src> <dst>` | 复制前询问是否覆盖（安全） |
| `cp -a <src> <dst>` | 归档模式复制（保留权限、时间戳、软链接） |
| `mv <src> <dst>` | 移动文件 / 重命名文件 |
| `mv -i <src> <dst>` | 移动前询问是否覆盖 |

### 文件查找

| 命令 | 说明 |
|:-----|:-----|
| `find . -name "*.txt"` | 当前目录及子目录中按名称查找 `.txt` 文件 |
| `find . -type f -mtime -7` | 查找最近 7 天内修改过的文件 |
| `find . -type d -name "node_modules"` | 查找名为 `node_modules` 的目录 |
| `find . -name "*.log" -exec rm {} \;` | 查找并删除所有 `.log` 文件 |
| `find . -size +100M` | 查找大于 100 MB 的文件 |
| `locate <keyword>` | 通过文件名数据库快速查找（先 `sudo updatedb`） |

### 文件权限

| 命令 | 说明 |
|:-----|:-----|
| `chmod 755 <file>` | 设置权限为 `rwxr-xr-x` |
| `chmod +x <file>` | 给文件添加可执行权限 |
| `chmod -R 755 <dir>` | 递归设置目录权限 |
| `chown <user>:<group> <file>` | 修改文件所属用户和组 |
| `chown -R <user>:<group> <dir>` | 递归修改所属用户和组 |

> [!TIP]- 权限数字速记
> - `r` = 4，`w` = 2，`x` = 1
> - `755` = 所有者 rwx + 组 r-x + 其他人 r-x
> - `644` = 所有者 rw- + 组 r-- + 其他人 r--
> - `777` = 所有人都有全部权限（不安全）

### 链接

| 命令 | 说明 |
|:-----|:-----|
| `ln -s <target> <link>` | 🔗 创建软链接（符号链接） |
| `ln <target> <link>` | 创建硬链接（共享同一 inode） |

---

## 📝 二、文本处理

### grep — 文本搜索

| 命令 | 说明 |
|:-----|:-----|
| `grep "pattern" <file>` | 在文件中搜索匹配的行 |
| `grep -i "pattern" <file>` | 忽略大小写搜索 |
| `grep -r "pattern" <dir>` | 递归搜索目录下所有文件 |
| `grep -v "pattern" <file>` | 反向搜索：显示**不匹配**的行 |
| `grep -n "pattern" <file>` | 显示匹配行的行号 |
| `grep -c "pattern" <file>` | 统计匹配行数 |
| `grep -A 3 -B 2 "<p>" <file>` | 显示匹配行 + 后 3 行 + 前 2 行 |
| `grep -E "a\|b" <file>` | 扩展正则（等同于 `egrep`） |

### 管道与重定向

| 命令 | 说明 |
|:-----|:-----|
| <code>cmd1 \| cmd2</code> | 管道：`cmd1` 的输出作为 `cmd2` 的输入 |
| `cmd > <file>` | 将输出重定向写入文件（**覆盖**原内容） |
| `cmd >> <file>` | 将输出**追加**到文件末尾 |
| `cmd < <file>` | 将文件内容作为命令的输入 |
| `cmd 2>&1` | 将 stderr 合并到 stdout |
| `cmd &> <file>` | 将 stdout + stderr 一起写入文件 |
| <code>cmd \| tee <file></code> | 同时在终端显示输出并写入文件 |

### 其他文本工具

| 命令 | 说明 |
|:-----|:-----|
| `sort <file>` | 按行排序 |
| `sort -n <file>` | 按数值排序 |
| `sort -u <file>` | 排序并去重 |
| `sort -t',' -k2 <file>` | 按逗号分隔后的第 2 列排序 |
| `uniq <file>` | 去除相邻重复行（通常先 `sort`） |
| `cut -d',' -f1 <file>` | 按逗号分隔，提取第 1 列 |
| `sed 's/old/new/g' <file>` | 全局替换文本（不改原文件） |
| `sed -i 's/old/new/g' <file>` | ⚠️ 直接修改原文件 |
| `awk '{print $1, $3}' <file>` | 打印第 1、3 列 |
| `diff <file1> <file2>` | 逐行比较两个文件的差异 |
| `xargs` | 将管道输入转为命令行参数 |

---

## 🗜️ 三、压缩与归档

| 命令 | 说明 |
|:-----|:-----|
| `tar -czf a.tar.gz <dir>` | 打包并用 **gzip** 压缩 |
| `tar -xzf a.tar.gz` | 解压 `.tar.gz` |
| `tar -cjf a.tar.bz2 <dir>` | 打包并用 **bzip2** 压缩（更小但更慢） |
| `tar -xjf a.tar.bz2` | 解压 `.tar.bz2` |
| `tar -xvf a.tar` | 解压纯 `.tar` 包 |
| `zip -r a.zip <dir>` | 压缩为 zip 格式 |
| `unzip a.zip` | 解压 zip 文件 |

> [!TIP]- tar 参数助记
> - `c` = create 创建，`x` = extract 解压
> - `z` = gzip，`j` = bzip2
> - `f` = file（**必须放最后**，后面跟文件名）
> - `v` = verbose 显示过程

---

## ⚙️ 四、进程管理

| 命令 | 说明 |
|:-----|:-----|
| `ps aux` | 列出系统所有进程 |
| <code>ps aux \| grep <name></code> | 按进程名筛选 |
| `top` | 实时进程资源监控（`q` 退出） |
| `htop` | 增强版 top，交互更友好（需安装） |
| `kill <PID>` | 优雅终止进程 |
| `kill -9 <PID>` | ⚠️ 强制杀死进程（不给清理的机会） |
| `killall <name>` | 按名称终止所有匹配进程 |
| `pkill -f <pattern>` | 按命令字符串模糊匹配终止 |
| `bg` / `fg` | 将任务放到后台 / 前台 |
| `jobs` | 列出当前会话的后台任务 |
| `nohup <cmd> &` | 后台运行，终端关闭后不挂断 |
| `Ctrl+Z` | 暂停当前前台任务 |
| `Ctrl+C` | 终止当前前台任务 |

---

## 💾 五、磁盘与内存

| 命令 | 说明 |
|:-----|:-----|
| `df -h` | 磁盘分区使用情况（人类可读） |
| `du -sh <dir>` | 目录的总大小 |
| `du -h --max-depth=1` | 当前目录下各子目录的大小 |
| `du -ah \| sort -rh \| head -10` | 🔥 找出最大的 10 个文件/目录 |
| `free -h` | 内存使用情况 |
| `lsblk` | 列出块设备（磁盘、分区） |
| `mount` / `umount` | 挂载 / 卸载文件系统 |
| `ncdu` | 交互式磁盘分析工具（需安装） |

---

## 🌐 六、网络操作

### 基本网络

| 命令 | 说明 |
|:-----|:-----|
| `ping <host>` | 测试连通性 |
| `curl <url>` | 发送 HTTP GET 请求 |
| `curl -O <url>` | 下载文件，保留原始文件名 |
| `curl -o <file> <url>` | 下载文件并指定文件名 |
| `curl -X POST -d 'data' <url>` | 发送 POST 请求 |
| `curl -H "key: val" <url>` | 添加自定义请求头 |
| `wget <url>` | 下载文件 |
| `wget -r <url>` | 递归下载整个网站 |

### 远程访问

| 命令 | 说明 |
|:-----|:-----|
| `ssh <user>@<host>` | SSH 远程登录 |
| `ssh -p <port> <user>@<host>` | 指定端口登录 |
| `ssh -i <key> <user>@<host>` | 使用私钥文件登录 |
| `scp <src> <user>@<host>:<dst>` | 通过 SSH 复制文件 |
| `scp -r <dir> <user>@<host>:<dst>` | 通过 SSH 复制整个目录 |
| `rsync -avz <src> <dst>` | 增量同步（本地或远程） |

### 端口与调试

| 命令 | 说明 |
|:-----|:-----|
| `ss -tlnp` | 查看正在监听的 TCP 端口 |
| `lsof -i :<port>` | 查看占用某端口的进程 |
| `ip addr` | 查看网络接口 IP 地址 |
| `nc -zv <host> <port>` | 检测 TCP 端口是否开放 |
| `nc -l <port>` | 在本机监听指定端口 |

---

## 👤 七、用户与权限

| 命令 | 说明 |
|:-----|:-----|
| `whoami` | 显示当前用户名 |
| `sudo <cmd>` | 以 root 权限执行命令 |
| `sudo su` | 切换到 root 用户 |
| `su <user>` | 切换到指定用户 |
| `passwd` | 修改当前用户密码 |
| `useradd -m <user>` | 添加新用户（并创建家目录） |
| `userdel -r <user>` | 删除用户（并删除家目录） |
| `usermod -aG <group> <user>` | 将用户添加到附加组 |

---

## 📦 八、软件包管理（Debian / Ubuntu）

| 命令 | 说明 |
|:-----|:-----|
| `sudo apt update` | 刷新软件包列表 |
| `sudo apt upgrade` | 升级所有可更新的包 |
| `sudo apt install <pkg>` | 安装软件包 |
| `sudo apt remove <pkg>` | 卸载软件包（保留配置文件） |
| `sudo apt purge <pkg>` | 彻底卸载（含配置文件） |
| `sudo apt autoremove` | 自动清理不再需要的依赖 |
| `apt search <keyword>` | 搜索软件包 |
| `dpkg -i <file>.deb` | 手动安装 `.deb` 包 |

---

## 🖥️ 九、系统信息

| 命令 | 说明 |
|:-----|:-----|
| `uname -a` | 系统内核和架构信息 |
| `lsb_release -a` | 发行版版本信息 |
| `hostnamectl` | 查看 / 修改主机名 |
| `uptime` | 系统运行了多久 |
| `date` | 显示 / 设置日期时间 |
| `cal` | 显示日历 |
| `history` | 查看命令历史 |
| `!<N>` | 执行历史中编号为 N 的命令 |
| `!!` | 重复上一条命令 |
| `alias ll='ls -la'` | 创建别名 |
| `which <cmd>` | 显示命令的完整路径 |
| `type <cmd>` | 显示命令的类型和来源 |

---

## 🔀 十、Git 操作

### 配置与初始化

| 命令 | 说明 |
|:-----|:-----|
| `git config --global user.name "..."` | 设置全局用户名 |
| `git config --global user.email "..."` | 设置全局邮箱 |
| `git init` | 初始化一个新的 Git 仓库 |
| `git clone <url>` | 克隆远程仓库 |
| `git clone -b <branch> <url>` | 克隆指定分支 |

### 基本工作流

| 命令 | 说明 |
|:-----|:-----|
| `git status` | 查看工作区、暂存区状态 |
| `git add <file>` | 将文件添加到暂存区 |
| `git add .` | 添加当前目录所有变更到暂存区 |
| `git add -A` | 添加所有变更（含删除） |
| `git add -p` | 交互式选择要暂存的代码块 |
| `git commit -m "msg"` | 提交并附上说明 |
| `git commit --amend` | 修改上一次提交 |
| `git push` | 推送本地提交到远程 |
| `git push -u origin <branch>` | 推送并建立分支追踪 |
| `git push --force-with-lease` | 安全强制推送 |
| `git pull` | 拉取远程更新并自动合并 |
| `git pull --rebase` | 拉取并以 rebase 方式合并 |
| `git fetch` | 拉取远程更新但**不合并** |
| `git log --oneline --graph --all` | 图形化展示分支历史 |
| `git log -p` | 查看提交详细 diff |
| `git blame <file>` | 查看文件每行的最后修改者 |

### 分支操作

| 命令 | 说明 |
|:-----|:-----|
| `git branch` | 列出本地分支 |
| `git branch -a` | 列出所有分支（含远程） |
| `git branch <name>` | 创建新分支 |
| `git branch -d <name>` | 删除已合并的分支 |
| `git branch -D <name>` | ⚠️ 强制删除分支 |
| `git checkout <branch>` | 切换分支 |
| `git checkout -b <branch>` | 创建并切换到新分支 |
| `git switch <branch>` | 切换分支（Git 2.23+） |
| `git switch -c <branch>` | 创建并切换（推荐代替 `checkout -b`） |
| `git merge <branch>` | 合并指定分支到当前分支 |
| `git merge --no-ff <branch>` | 禁用快进合并，保留分支痕迹 |
| `git rebase <branch>` | 将当前分支变基到目标分支 |
| `git rebase -i HEAD~N` | 交互式变基（合并/修改/排序最近 N 次提交） |
| `git cherry-pick <commit>` | 将某次提交应用到当前分支 |

### 撤销与回退

| 命令 | 说明 |
|:-----|:-----|
| `git restore <file>` | 撤销未暂存的修改 |
| `git restore --staged <file>` | 取消暂存（回到工作区） |
| `git reset --soft HEAD~1` | 撤销提交，变更保留在**暂存区** |
| `git reset --mixed HEAD~1` | 撤销提交，变更保留在**工作区** |
| `git reset --hard HEAD~1` | ⚠️ 撤销提交并**丢弃所有变更** |
| `git reset --hard origin/main` | 将本地重置为远程 main 分支状态 |
| `git revert <commit>` | 新建一个提交来撤销指定提交（✅ 安全） |
| `git stash` | 暂存当前工作区变更 |
| `git stash pop` | 恢复最近一次暂存 |
| `git stash list` | 查看 stash 列表 |
| `git clean -fd` | ⚠️ 删除未追踪的文件和目录 |

### 远程仓库 & 标签

| 命令 | 说明 |
|:-----|:-----|
| `git remote -v` | 查看远程仓库地址 |
| `git remote add <name> <url>` | 添加远程仓库 |
| `git remote set-url <name> <url>` | 修改远程仓库地址 |
| `git tag -a v1.0 -m "msg"` | 创建附注标签 |
| `git push --tags` | 推送所有标签到远程 |
| `git tag -d v1.0` | 删除本地标签 |

> [!TIP]- Git 回退场景速查
> - 只想撤销**还没 commit** 的改动 → `git restore <file>` 或 `git restore --staged <file>`
> - 想撤销**最近一次 commit** 但保留改动 → `git reset --soft HEAD~1`
> - 想撤销**已推送的 commit** 且不影响他人 → `git revert <commit>`
> - 脑子全乱了想回到远程版本 → `git reset --hard origin/main`
> - 丢失了 commit 想找回 → `git reflog` 看看历史

---

## ⌨️ 十一、终端快捷键

| 快捷键 | 说明 |
|:-------|:-----|
| `Tab` | 自动补全命令 / 路径 / 文件名 |
| `Ctrl + C` | 终止当前运行的命令 |
| `Ctrl + Z` | 暂停当前前台任务 |
| `Ctrl + D` | 发送 EOF（退出当前 shell） |
| `Ctrl + L` | 清屏 |
| `Ctrl + R` | 🔍 搜索命令历史记录 |
| `Ctrl + A` | 光标移到行首 |
| `Ctrl + E` | 光标移到行尾 |
| `Ctrl + U` | 删除光标前所有内容 |
| `Ctrl + K` | 删除光标后所有内容 |
| `Ctrl + W` | 删除光标前一个单词 |
| `Alt + B` / `Alt + F` | 光标向前 / 向后移动一个单词 |
| `!!` | 重复上一条命令 |

---

## 🧩 十二、实用组合示例

```bash
# ── 磁盘相关 ──────────────────────────
# 找出占用磁盘最大的 10 个文件 / 目录
du -ah /path | sort -rh | head -10

# 查找大于 100MB 的文件
find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null

# ── 文本搜索 ──────────────────────────
# 递归搜索代码中的 TODO 注释
grep -r "TODO" --include="*.py" --include="*.js" .

# 统计某关键词在代码中出现次数
grep -r "function" --include="*.js" . | wc -l

# ── 端口与进程 ─────────────────────────
# 杀死占用某端口的进程
sudo lsof -ti:3000 | xargs kill -9

# 查看内存占用前 10 的进程
ps aux --sort=-%mem | head -11

# ── 文件处理 ──────────────────────────
# 统计代码行数（排除 node_modules）
find . -name "*.js" -not -path "*/node_modules/*" | xargs wc -l

# 批量重命名：所有 .txt → .md
for f in *.txt; do mv "$f" "${f%.txt}.md"; done

# ── Git 相关 ──────────────────────────
# 查看提交最多的贡献者
git shortlog -sn --all

# 列出本地修改过的文件
git diff --name-only HEAD~1

# ── 远程同步 ──────────────────────────
# 通过 rsync 增量同步
rsync -avz --progress user@host:/remote/dir/ /local/dir/
```
