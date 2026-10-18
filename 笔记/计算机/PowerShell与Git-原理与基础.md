# PowerShell 与 Git：原理与基础

> **AI 辅助生成**
>
> 本文由 AI 于 2026-10-18 整理成稿，**不是本人原创撰写**。文中的案例、报错与命令行输出均来自本机的真实操作记录（装 Git、PATH 冲突、`git rm` 删空目录等），但组织方式与解释由 AI 完成。
> 若读下来与你的实测不一致，以实测为准，并回来修正本文。

> 一句话定位：从「终端里敲的那行字」开始，讲清这两套系统底层的模型，而不是罗列命令。

---

## 一、先分清三样东西：终端、shell、命令

初学者最大的困惑来自把这三层混成一件事。它们各管一段，职责完全不同。

| 层 | 是什么 | 职责 |
|---|---|---|
| **终端 Terminal** | 窗口本身（Windows Terminal、VS Code 终端面板） | 只负责显示字符、把键盘输入交给 shell。它不理解你敲的是什么 |
| **Shell** | 解释器（bash、PowerShell、zsh） | 真正「读懂」这句话的程序。它决定 `\|` 是管道、`$x` 是变量、`for` 怎么循环 |
| **命令 Command** | 磁盘上的可执行文件（`git.exe`、`ls`、`python.exe`） | shell 自己通常不实现它们，只负责启动 |

一条命令的执行链路：

```
你输入 → 终端 → shell 解析 → 按 PATH 找到程序 → 启动进程 → 输出回终端
```

这个分层解释了三个常见误解：

- **换终端不等于换 shell。** Windows Terminal 里可以开 PowerShell，也可以开 Git Bash。窗口变了，解释器可能没变。
- **bash 和 PowerShell 不是「两个命令」。** 它们是同一层的两个竞品，都是 shell。
- **`ls` 不是一个程序，是很多个。** Linux 的 `ls` 是 `/usr/bin/ls`；PowerShell 里敲 `ls`，实际执行的是 `Get-ChildItem`，一个完全不同的东西。同名不同物。

---

## 二、命令是怎么被找到的：PATH 的原理

你敲 `git`，shell 不会去全盘搜索。它只做一件事：

> **按 `PATH` 环境变量里列出的目录顺序逐个查找同名可执行文件，找到第一个就用。**

```bash
# bash：冒号分隔
echo $PATH | tr ':' '\n'
```

```powershell
# PowerShell：分号分隔
$env:Path -split ';'
```

关键在于三个字：**第一个**。一旦命中，后面目录里就算有同名程序也不会被使用。

> [!WARNING]
> **真实案例（本机 2026-10-18 发生过）**
>
> 敲 `git` 报「不是内部或外部命令」，因为当时机器上根本没装 Git，只有一个临时借用的便携版。
>
> 后来装了官方 Git 到 `E:\Git`，它写进的是**系统 PATH**；而临时那个便携版在**用户 PATH**。Windows 拼接 PATH 时是**用户在前、系统在后**，于是旧的便携版一直先被命中，装了新的像没装一样。
>
> 删掉用户 PATH 里那条之后才真正生效。**这就是「PATH 顺序即优先级」的实例。**

还有一个 Windows 特有的坑：**PATH 改动只对新开的进程有效**。已在运行的终端不会自动刷新，所以改完必须重开窗口。这也是「改完没反应」最常见的原因。

> [!TIP]
> **排查口诀**：命令找不到时，问三个问题 —— ① 这程序真的装了吗？② 它在 PATH 里吗？③ PATH 里有没有更靠前的同名项抢走了？

---

## 三、PowerShell 的原理

### 3.1 核心差异：管道里流的是对象，不是文本

这是 PowerShell 与 bash 最本质的区别，理解这一条就够了。

bash 的管道传的是**字符串**，想从进程列表里挑出 python，必须把文本切开再匹配：

```bash
# bash：什么都得靠文本处理
ps aux | grep python | awk '{print $2}'
```

PowerShell 的管道传的是 **.NET 对象**。进程本身就是对象，有 `Name`、`Id`、`CPU` 这些属性，直接取就行：

```powershell
# PowerShell：直接操作属性
Get-Process python | Select-Object Id, CPU

# 按属性筛选，不用解析文本
Get-Process | Where-Object CPU -gt 100 | Sort-Object CPU -Descending
```

为什么这很重要？因为 bash 里 `grep python` 会误伤——只要某行的**任意位置**出现 "python" 就算命中，比如某个路径里带 python 的进程。而 PowerShell 按属性比较，是精确的。

对应链路对比：

| | 链路 |
|---|---|
| bash | `程序` → 输出**文本** → `grep/awk` 解析 → 文本 |
| PowerShell | `Get-Process` → **对象集合** → `Where-Object` → 对象集合 → `Format-Table` |

PowerShell 把「解析文本」这一步省掉了，因为对象天生带结构。

### 3.2 命令命名：动词-名词

PowerShell 的命令叫 **cmdlet**，格式固定为 `动词-名词`，动词受官方列表约束（Get / Set / New / Remove / Start / Stop / Test / Invoke…）。好处是可猜、可发现：

```powershell
Get-Command *process*              # 列出所有和 process 有关的命令
Get-Help Get-Process -Examples     # 看用法
Get-Member                         # 接在管道后，看对象到底有哪些属性
```

`Get-Member` 是 PowerShell 里最该学会的一条命令：它告诉你「手上这个对象能取什么」，等于对象版的 `man`。

### 3.3 别名陷阱：ls 不是你以为的 ls

| 你敲的 | 实际执行 | 注意 |
|---|---|---|
| `ls` | `Get-ChildItem` | 输出是表格对象，不是彩色列表 |
| `cat` | `Get-Content` | 返回行数组，不是字节流 |
| `rm` | `Remove-Item` | **没有回收站，删了就是删了** |
| `cp` / `mv` | `Copy-Item` / `Move-Item` | 参数名是 `-Destination`，不是 `-d` |
| `ps` | `Get-Process` | 和 Linux 的 `ps` 完全不同 |

查全部别名用 `Get-Alias`；确认某个名字到底指向谁，用 `Get-Command ls`。

**别名只是名字相同，语义和行为都可能不一样**——这是从 bash 转到 PowerShell 最容易吃亏的地方。

### 3.4 变量、环境变量与类型

```powershell
# 普通变量（自动推断类型）
$name = "learning-log"

# 环境变量有专门的驱动器前缀 $env:
$env:Path
$env:http_proxy = $null          # 等价于 bash 的 unset

# 强类型
[int]$n = "42"
```

> [!NOTE]
> **为什么 `unset` 在 PowerShell 里报错？**
>
> 因为 `unset` 不是程序，而是 **bash 这门语言的关键字**。你在 PowerShell 里敲它，PowerShell 会当成「去 PATH 里找一个叫 unset.exe 的程序」，找不到就报「无法识别」。
>
> 同理 `cd /d D:\x` 里的 `/d` 是 cmd 的参数，PowerShell 不认。
> **shell 的语法差异不是用法差异，是语言差异。**

### 3.5 执行策略：脚本能不能跑

`ExecutionPolicy` 控制本机是否允许运行 `.ps1` 脚本，默认通常是 `Restricted`（全禁）。它**不是安全边界**，只是防误点：

```powershell
Get-ExecutionPolicy                                    # 查当前
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned    # 只放开当前用户
```

顺带记住 `$PROFILE`：这个文件里的内容会在每次启动 PowerShell 时自动执行，适合放别名和提示符设置。路径用 `echo $PROFILE` 查。

### 3.6 PowerShell 5.1 与 7 是两回事

| | Windows PowerShell 5.1 | PowerShell 7+ |
|---|---|---|
| 运行时 | .NET Framework | .NET（跨平台） |
| 可执行文件 | `powershell.exe` | `pwsh.exe` |
| 平台 | 仅 Windows | Windows / Linux / macOS |
| 状态 | 只修 bug，不加功能 | 持续开发 |

很多教程不区分这两者，导致命令抄过来报错。查版本用 `$PSVersionTable`。

---

## 四、Git 的原理

### 4.1 最重要的一句：Git 存的是快照，不是差异

多数版本控制工具（SVN、CVS）存的是「每次改了哪几行」。Git 不这么干：

> **每次提交，它为当时所有文件各存一份完整内容。**

听起来很浪费，但有个关键设计让它不浪费：**内容相同就只存一份**。因为每个对象都用内容的哈希值命名，内容没变的文件哈希没变，直接复用上一个版本的那一份。

Git 的特点基本都是这个设计推出来的：

- **切分支几乎瞬间完成** —— 只是移动一个指针，不用改文件
- **能离线看完整历史** —— 所有版本本来就在本地
- **任何提交都能完整还原** —— 那是当时所有文件的真实内容，不是「往前推演」

### 4.2 四种对象：blob、tree、commit、tag

| 对象 | 是什么 | 类比 |
|---|---|---|
| **blob** | 文件的内容（不含文件名） | 一份文件正文 |
| **tree** | 一个目录：列出子项的名字、权限、指向哪个 blob 或 tree | 目录清单 |
| **commit** | 指向一个 tree，加上父提交、作者、时间、提交信息 | 一张写着「第 N 版」的封条 |
| **tag** | 给某个 commit 起的别名（如 `v1.0`） | 书签 |

每个对象的文件名就是它内容的 SHA-1 哈希（40 位十六进制），存放在 `.git/objects/` 下，路径是前两位做目录、后 38 位做文件名。可以自己看：

```bash
git cat-file -p HEAD            # 看当前提交对象
git cat-file -p HEAD^{tree}     # 看它指向的目录树
git rev-parse HEAD              # 看这个提交的完整哈希
```

> [!NOTE]
> **为什么内容一改哈希就全变？**
>
> 因为哈希是**内容算出来的**。改一个字，哈希完全不同，父提交的 tree 就变了，再上一层 commit 也变。这条链保证了历史**无法被悄悄篡改**——改了任何一处，后面的哈希全对不上。

### 4.3 三棵树：工作区、暂存区、版本库

Git 最容易被绕晕的地方。记住：**你的文件同时可能存在于三个地方**。

```
  工作区                暂存区               版本库
D:\code\...   --add-->  .git/index  --commit-->  .git/objects
（随便改）            （待打包清单）          （真正的历史）
```

| 区域 | 是什么 | 在哪 |
|---|---|---|
| **工作区** | 你眼睛看到的文件夹，随便改，Git 不管 | `D:\code\learning-log\` |
| **暂存区** | 「下次提交要包含什么」的清单 | `.git/index` |
| **版本库** | 真正的历史，只追加、不改写 | `.git/objects` |

理解了三个区，就能看懂两个 `diff` 的区别——很多人用了半年还搞不清：

| 命令 | 比的是谁和谁 | 回答的问题 |
|---|---|---|
| `git diff` | 工作区 ↔ 暂存区 | 我改了但还没 add 的有什么 |
| `git diff --cached` | 暂存区 ↔ 上次提交 | 我 add 了准备提交的有什么 |
| `git status` | 三者综合 | 整体概览 |

### 4.4 分支只是一个 41 字节的文本文件

这是 Git 最反直觉的一点：**分支不是文件夹副本，只是一个存着哈希的文件。**

```bash
cat .git/refs/heads/main     # 输出 40 位哈希
cat .git/HEAD                # 输出 ref: refs/heads/main
```

所以：

- **`git switch` 为什么瞬间完成？** 它只改 HEAD 指向，然后把工作区文件换成目标提交的内容。
- **为什么建分支几乎零成本？** 建一个新文件，写一个哈希进去，完事。
- **为什么叫「切换」不叫「复制」？** 因为从来没有第二份文件树，只有一份，随时被换掉。

多个提交靠「父提交」指针连起来，形成一张**有向无环图**。合并（merge）的本质，就是造一个新提交，让它有**两个父提交**。

### 4.5 远端：只是另一份仓库

Git 是分布式的，**没有「服务器」这个概念，只有「另一个仓库」**。`origin` 只是给某个远端 URL 起的默认名字，不特殊。

| 命令 | 实际做了什么 |
|---|---|
| `clone` | 把对方的整个对象库复制过来，再建本地分支 |
| `fetch` | 只下载对方有、我没有的对象，更新 `origin/main` 这个记录。**不动你的文件** |
| `merge` | 把 `origin/main` 合进当前分支 |
| `pull` | `fetch` + `merge`（或 rebase），一步做完 |
| `push` | 把我多出来的对象传上去，再请求对方移动它的分支指针 |

> [!WARNING]
> **为什么 push 会被拒绝？**
>
> 因为 push 本质是「请你把 main 指到我这个提交」。如果对方的 main 上有你没有的提交，指过去就会**丢掉那些提交**，所以 Git 会拒绝，让你先 fetch 合并。
>
> 这不是权限问题，是**数据完整性保护**。这也是为什么 `git push --force` 危险：它跳过这个保护，强行覆盖。

还有一个容易混的：`origin/main` 不是「服务器上的分支」，而是**你本地记着的、上次 fetch 时服务器长什么样**。它可能已经过期了。真正去问服务器，要用 `git fetch` 或 `git ls-remote`。

### 4.6 为什么 .git 目录那么大

因为它装着**全部历史**。一个几百 MB 的仓库，`.git` 可能比工作区还大很多——里面的每个文件、每个版本都在。

这也是为什么不该把大文件、数据集、视频提交进去：它们会被永久留在历史里，`clone` 一次就要全下载下来。

---

## 五、两套体系如何配合

它们不在同一层：**PowerShell 是 shell（执行环境），Git 是被它调用的程序**。所以关系是「谁启动谁」，不是「谁替代谁」。

```
你输入  git commit -m "补 10-19"
  ↓
PowerShell 解析 → 拆出程序名和参数 → 按 PATH 找到 E:\Git\cmd\git.exe
  ↓ 启动进程
git.exe 读 .git/ → 生成 tree 和 commit → 移动 main 指针 → 打印结果
  ↓
PowerShell 把输出交给终端显示
```

所以两种用法并存：

- **PowerShell 负责环境**：装没装、PATH 对不对、代理变量要不要清、文件在不在。
- **Git 负责内容**：提交什么、分支怎么走、推到哪。

---

## 六、四个真实踩过的现象，背后的原理

这一段是全文最实用的部分，因为每个现象都真实发生过。

### 现象 1：`git` 提示「不是内部或外部命令」

**原理**：shell 按 PATH 找可执行文件，找不到就报这个错。当时机器上确实没装 Git，只有一个工具自带的便携版，而它不在 PATH 里。

**教训**：「命令不存在」永远先查 PATH，而不是怀疑装坏了。

### 现象 2：`unset` 和 `cd /d` 都报错

**原理**：它们是 bash / cmd 的**语法**，不是程序。PowerShell 把它们当程序名去找，自然找不到。

```bash
# bash
unset http_proxy
```

```powershell
# PowerShell
$env:http_proxy = $null
```

```
:: cmd
set http_proxy=
```

### 现象 3：`git rm 项目/README.md` 之后，整个「项目」文件夹消失了

**原理**：Git 的基本规则 —— **它只跟踪文件，不跟踪目录**。目录之所以存在，是因为里面有被跟踪的文件。`tree` 对象里没有「空目录」这种条目，它的每个条目都必须指向一个 blob 或 subtree。

**所以**：删掉目录里最后一个被跟踪的文件，目录随之消失。想保留空目录，必须放一个占位文件——这就是 `.gitkeep` 存在的原因（它没有任何特殊含义，只是「非空」的借口）。

### 现象 4：文件列表右边出现了「说明」

**原理**：那不是文件描述，是**最后一次改动这个路径的 commit message**。GitHub 的文件浏览器固定显示这一列，用来显示「这个文件上次什么时候被动过」。同一次提交改了多个文件夹，它们的右侧就会显示同一句话。

**这也解释了另一件事**：为什么子目录里的 `README.md` 会「漏出来」。因为 GitHub 会把目录中的 `README.md` 当作该目录的首页渲染。这不是漏洞，是约定。所以内部说明别放子目录。

---

## 七、速查表

### 7.1 bash 与 PowerShell 对照

| 想干什么 | bash | PowerShell |
|---|---|---|
| 看目录 | `ls -la` | `Get-ChildItem -Force` |
| 换目录 | `cd /d/code` | `Set-Location D:\code` |
| 看文件内容 | `cat f.txt` | `Get-Content f.txt` |
| 搜文本 | `grep -rn "x" .` | `Select-String -Path * -Pattern x -Recurse` |
| 找文件 | `find . -name "*.md"` | `Get-ChildItem -Recurse -Filter *.md` |
| 看进程 | `ps aux` | `Get-Process` |
| 杀进程 | `kill -9 PID` | `Stop-Process -Id PID -Force` |
| 设环境变量 | `export X=1` | `$env:X = 1` |
| 查命令位置 | `which git` | `Get-Command git` |
| 看帮助 | `man git` | `Get-Help git` |

### 7.2 Git 日常最少要会的

| 命令 | 作用 | 在哪一层操作 |
|---|---|---|
| `git status` | 看三棵树的状态 | 只读 |
| `git add -A` | 全部改动进暂存区 | 工作区 → 暂存区 |
| `git commit -m "..."` | 固化为一个版本 | 暂存区 → 版本库 |
| `git log --oneline` | 看历史，一行一条 | 只读 |
| `git diff` | 看没 add 的改动 | 只读 |
| `git push` | 传到远端 | 版本库 → 远端 |
| `git pull` | 拉下来并合并 | 远端 → 本地 |
| `git restore <file>` | 丢弃工作区改动 | 覆盖工作区 |
| `git revert <commit>` | 造一个新提交来抵消旧提交 | 追加，不改历史 |

> [!WARNING]
> **一条必须记住的红线：** `git reset --hard` 会直接丢弃未提交的改动，且**无法恢复**。在没搞清楚三棵树之前不要用它。想撤销已推送的提交，用 `revert`，不要用 `reset` + 强推。

---

## 八、接下来怎么学

按这个顺序，每一步都有可验证的产出：

1. **把三棵树走一遍**（半小时）。随便改一个文件，依次跑 `git status` → `git add` → `git status` → `git diff --cached`，观察每个阶段的变化。这一步比读十篇文章都管用。
2. **亲手看一次对象**（十分钟）。跑 `git cat-file -p HEAD`，就能看到自己刚才那次提交长什么样：一个 tree 哈希、一个父提交、一行信息。抽象概念会立刻落地。
3. **犯一次错并被救回来**。改乱一个文件，用 `git restore` 恢复；提交错了信息，用 `git commit --amend` 改。被救回来一次，就真的记住了。
4. **再碰分支**。等需要「同时写两件不同的事」时再学分支，那时候它有具体用途，不会变成背命令。

> [!TIP]
> 这个阶段最该做的不是学更多命令，而是**每天真的用一次**。`add / commit / push` 三条命令连用两周，比读完这篇文章更有价值。
> 原理是用来解释困惑的，不是用来背的——遇到「为什么会这样」时再回来翻对应章节。
