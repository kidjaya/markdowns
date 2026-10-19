# Git · 核心知识清单

> 面向：5 年经验 Java 后端工程师，从"会用"走向"懂原理"
> 用法：当作自检清单，答不出来的条目就是下一步要补的地方

## 一、核心心智模型

1. **Git 存的是快照，不是差异。** 每次 commit 都是整棵目录树的一个完整快照（未改动的文件复用之前的 blob 对象）。这不是浪费——内容寻址 + 硬链接式的对象复用让"全量快照"的成本接近"增量"。理解这一点，分支、merge、rebase 的行为才解释得通。
2. **分支只是一个指向 commit 的 40 字节指针。** 所以创建分支是 O(1) 的，切换分支本质是移动 HEAD 指针 + 按快照重建工作区。所有"分支很重"的直觉都来自于 SVN。
3. **内容寻址是 Git 一切特性的地基。** 对象的 SHA-1 由内容决定，内容相同则 ID 相同。因此：去重是免费的、完整性校验是免费的（改一个字节 ID 就变了）、分布式同步不需要中心权威。
4. **三个区域（工作区 / index / 版本库）之间的四种转移操作，就是 Git 的全部动词。** `add` `commit` `checkout` `reset` 都是在这三区之间搬东西，只是方向不同。把这张图刻在脑子里，就不会搞混 `reset --hard` 和 `checkout`。
5. **reflog 是所有"撤销"操作的最后一道防线。** 只要 commit 对象还在（未 gc），任何 HEAD 的移动都被记录在 `.git/logs/HEAD`。所以 Git 里几乎没有真正不可逆的操作——除非你手动 gc 或者从没提交过。

## 二、必懂机制

### 2.1 底层对象模型

四类对象：

| 对象 | 存什么 | 生成时机 |
|---|---|---|
| **blob** | 文件内容（不含文件名） | `git add` 时 |
| **tree** | 目录结构，记录 文件名 → blob/tree 的映射 + 权限位 | `git commit` 时 |
| **commit** | 指向一个 tree + 父 commit 的 SHA + author/committer + 时间戳 + message | `git commit` 时 |
| **tag** | 指向某个 commit（轻量 tag）或含签名信息的 tag 对象 | `git tag` |

一次 commit 的完整生成过程：
```
工作区改动 → git add → 为每个文件算 SHA-1 生成 blob 存入 .git/objects
                     → 把 blob 的 SHA 写进 index（暂存区）
           → git commit → 用 index 内容生成 tree 对象（递归生成子目录 tree）
                        → 生成 commit 对象指向该 tree + 当前 HEAD 指向的 commit 作为父
                        → 把 commit 的 SHA 写进当前分支的 ref
```

**关键引用**：
- `HEAD` —— 通常指向当前分支的 ref（`refs/heads/main`），`git checkout <commit>` 后进入 detached HEAD 直接指向 commit
- `ORIG_HEAD` —— 危险操作（reset/merge/rebase）前的 HEAD 位置，兜底用
- `FETCH_HEAD` —— 最近一次 `git fetch` 拿到的远端分支位置
- `refs/remotes/origin/xxx` —— 远端跟踪分支，本地缓存的远端状态，**不是实时的**

**packfile 与 gc**：松散对象多了之后，`git gc` 会把它们打包成 packfile 并做 delta 压缩（只存差异）。`git push` 前自动打包就是这个原理。**已被删除引用又超过 reflog 保留期的对象会被 gc 回收**——这是"误操作后还能不能救"的时间窗口。

```bash
git cat-file -p <sha>      # 查看任意对象的原始内容（blob/tree/commit 都行）
git cat-file -t <sha>      # 查看对象类型
git rev-parse HEAD         # 把 ref 解析成完整 SHA
git rev-parse --short HEAD # 短 SHA
```

### 2.2 撤销三件套的精确差异（最容易被搞混的地方）

假设文件已修改且已 `git add`：

| 命令 | HEAD | index | 工作区 | 语义 |
|---|---|---|---|---|
| `git restore <file>` | 不动 | 不动 | **从 index 恢复** | 丢弃工作区改动（回到 add 后的状态） |
| `git restore --staged <file>` | 不动 | **从 HEAD 覆盖** | 不动 | 取消暂存 |
| `git restore --source=HEAD --staged --worktree <file>` | 不动 | 从 HEAD 覆盖 | 从 HEAD 覆盖 | 彻底还原到 HEAD |
| `git checkout -- <file>` | 不动 | 从 HEAD 覆盖 | **从 index 恢复** | 老式写法，语义模糊，Git 2.23+ 不推荐 |
| `git reset HEAD <file>` | 不动 | 从 HEAD 覆盖 | 不动 | 等价于 `restore --staged` |

**`reset` 三模式**（HEAD 指针 + index + 工作区，三个层级的移动范围）：

```bash
git reset --soft  HEAD~1   # 只移动 HEAD。改动仍在 index 和 工作区 → "我后悔 commit 了，想改成一次提交"
git reset --mixed HEAD~1   # 默认。移动 HEAD + 重置 index。改动留在工作区但未暂存 → "我后悔 add 了"
git reset --hard  HEAD~1   # 三个都动。工作区改动直接消失 → 唯一会丢代码的模式
```

**`revert` vs `reset` 的取舍**（面试必考）：

| | revert | reset |
|---|---|---|
| 历史 | **新增一个 commit**，原 commit 保留 | 移动指针，后续 commit 从历史中"消失" |
| 已推送的公共分支 | ✅ 安全，其他人 pull 后自动获得反向改动 | ❌ 改写历史，其他人 pull 会冲突/diverge |
| 能否撤销 | 可以 `revert` 那个 revert commit | 靠 reflog |
| 适用场景 | 公共分支回滚 | 本地未推送的提交 |

**一句话原则：已经 push 的就用 revert，还没 push 的才能 reset。**

### 2.3 reflog —— 唯一的后悔药

```bash
git reflog                    # 等价于 git reflog show HEAD
git reflog show feature/login # 看某个分支的移动记录
git reset --hard HEAD@{3}     # 回到 3 步之前
git checkout -b rescue HEAD@{2}  # 从某个历史位置新建分支保住代码
```

- 记录位置：`.git/logs/HEAD` 和 `.git/logs/refs/heads/<branch>`
- 默认保留 90 天（`gc.reflogExpire`），未被任何 ref 引用的对象 30 天后可回收（`gc.reflogExpireUnreachable`）
- **`git fsck --lost-found` 是最后兜底**：reflog 被清掉后，用它找 dangling commit

经典救援场景（每个都要能背下来）：
```bash
# 场景：git reset --hard 之后发现删错了
git reflog                      # 找到 reset 之前的 commit SHA
git reset --hard <sha>

# 场景：误删了未合并的分支
git reflog                      # 找分支 tip 的 SHA
git branch feature/recovered <sha>

# 场景：rebase 搞乱了想重来
git rebase --abort

# 场景：detached HEAD 下提交了代码然后 checkout 走了
git reflog                      # 找到那个 commit
git checkout -b save-work <sha>
```

### 2.4 分支、merge 与 rebase

**merge 的两种模式**：
- **fast-forward**：目标分支是源分支的祖先，直接把指针往前挪，不产生新 commit。缺点是丢失"这里合并过 feature 分支"的信息
- **`--no-ff`**：强制产生一个 merge commit。推荐在特性分支合入主干时使用，保留分支脉络，`git log --graph` 才读得懂，也方便整体回滚

**merge vs rebase 的本质差异**：

| | merge | rebase |
|---|---|---|
| 历史 | 保留分叉，真实记录发生过什么 | 把提交"重放"到目标分支顶端，历史变线性 |
| commit SHA | 不变 | **全部改变**（因为父 commit 变了） |
| 冲突解决 | 一次解决一个大冲突 | 可能要解决 N 次（每个 commit 一次） |
| 公共分支 | ✅ 安全 | ❌ 禁止 |

**绝对不要在已推送的公共分支上 rebase** —— 你本地重放后 SHA 全变了，别人基于旧 SHA 的提交会 diverge，最终强行 push 会把同事的提交抹掉。

**`rebase -i` 的常用动作**：
```
pick   保留该 commit
reword 保留但改 message
edit   停在这个 commit，允许修改内容
squash 合并到前一个 commit，保留两条 message
fixup  合并到前一个 commit，丢弃本条 message
drop   删除该 commit
```
还可以直接**调整行顺序**来重排提交（重排可能引发冲突，因为后面的改动依赖前面的）。

```bash
git rebase -i HEAD~3                    # 整理最近 3 个提交
git rebase -i --root                    # 从第一个提交开始整理
git cherry-pick <sha>                   # 摘一个提交到当前分支
git cherry-pick -n <sha>                # 只应用改动不提交（可多个合并成一次）
git cherry-pick A..B                    # 摘一个区间（不含 A）
git rebase --onto main feature oldBase  # 把 feature 上从 oldBase 之后的提交移植到 main 上
```

### 2.5 stash 与 clean

```bash
git stash push -m "临时保存：xx功能改了一半"   # 强烈建议带 message
git stash list
git stash pop              # 恢复并删除（可能冲突）
git stash apply            # 恢复但保留记录（更安全，可多次应用）
git stash apply stash@{1}  # 恢复指定的
git stash branch newbr     # 从 stash 直接建分支（冲突时的优雅解法）
git stash -u               # 关键：默认不含未跟踪文件，加 -u 才包含
git stash -a               # 连 .gitignore 忽略的文件也存（慎用）
git clean -fd              # 删除未跟踪的文件和目录
git clean -fdx             # 危险：连 ignored 的也删（会删掉 target/ node_modules/）
git clean -fdn             # dry-run，先看看会删什么
```

### 2.6 远程协作

```bash
git fetch origin              # 只拉取远端状态到本地 refs/remotes，不改工作区
git pull                      # = fetch + merge
git pull --rebase             # = fetch + rebase，避免无意义的 merge commit（推荐）
git config --global pull.rebase true   # 全局默认（推荐设置）
git branch -u origin/feature  # 设置上游跟踪
git push -u origin feature    # 首次推送并设置跟踪
git push --force-with-lease   # 唯一安全的强推：检查远端 ref 是否还是你 fetch 时的值
git push origin --delete feature  # 删远端分支
git fetch -p                  # prune，清理本地残留的远端已删分支引用
git remote prune origin       # 同上
```

**`--force` vs `--force-with-lease`**：
- `--force`：无条件覆盖远端。如果同事在你 fetch 之后推了新提交，会被你静默抹掉
- `--force-with-lease`：推送前检查远端 ref 的当前值是否等于本地记录的 `refs/remotes/origin/xxx`。不等则拒绝。**这是唯一应该使用的强推方式**
- 变体 `--force-with-lease=feature`：显式指定期望值，更严格

**`fetch` vs `pull`**：`fetch` 是"只更新我本地的远端快照"，不会动你的工作区，可以先 `git log origin/main..main` 看看差异再决定。`pull` 直接合并，容易被动接受冲突。**推荐习惯：先 fetch，看清楚，再决定 merge 还是 rebase。**

### 2.7 diff 的双点与三点（易错）

```bash
git diff A..B      # B 相对 A 的差异（A 有 B 没有 + B 有 A 没有）→ 等价于 git diff A B
git diff A...B     # 从 A 和 B 的公共祖先开始，到 B 的差异 → 只看 B 这一侧独有的改动
git diff --cached  # index vs HEAD（已暂存的改动）
git diff HEAD      # 工作区 + index vs HEAD（所有未提交的改动）
```

典型用法：`git diff main...feature` 看"feature 分支从拉出来到现在改了什么"——这是 PR 里看到的正确 diff。

### 2.8 排查神器：blame 与 bisect

```bash
git blame -L 100,120 OrderService.java    # 看 100~120 行每行的最后修改者与提交
git blame -w -L 100,120 file              # -w 忽略空白改动（避免被格式化提交污染）
git log -S "userBalance" --oneline        # 找哪些提交增删了某个字符串（比 blame 更强）
git log -S "foo" --pickaxe-regex -- '*.java'
```

**bisect 完整流程**（二分定位引入 bug 的提交，5 年经验必须会）：
```bash
git bisect start
git bisect bad                 # 当前 HEAD 是坏的
git bisect good v1.2.0         # 某个已知的好版本
# Git 自动 checkout 到中间的 commit
# 你跑测试/验证
git bisect bad   或  git bisect good
# 重复若干轮（log2(N) 次）
git bisect reset               # 结束，回到起点
```
全自动版：
```bash
git bisect start HEAD v1.2.0
git bisect run mvn test -Dtest=OrderTest   # 用测试脚本自动判定，退出码 0=good
git bisect reset
```

### 2.9 配置

三级配置，优先级：**local > global > system**
```bash
git config --system   # /etc/gitconfig（或 Git 安装目录）
git config --global   # ~/.gitconfig
git config --local    # .git/config（仓库级，默认）
git config --list --show-origin   # 看每个配置来自哪个文件（排障神器）
```

**CRLF —— Windows 上最大的坑**：
```bash
git config --global core.autocrlf true    # Windows 推荐：检出转 CRLF，提交转 LF
git config --global core.autocrlf input   # Linux/Mac：提交转 LF，检出不变
git config --global core.autocrlf false   # 关闭转换
git config --global core.safecrlf warn    # 混合换行时警告
```
- **坑的表现**：没设 autocrlf，Windows 同事提交的文件全是 CRLF，Linux 同事检出后 `git diff` 显示整个文件被改动（其实是换行符变了），code review 全是噪音
- **根治方案**：仓库根目录放 `.gitattributes`：
  ```
  * text=auto eol=lf
  *.java text eol=lf
  *.sh text eol=lf
  *.bat text eol=crlf
  *.png binary
  ```
  `.gitattributes` 覆盖本地 autocrlf 设置，团队成员行为一致
- **另一个坑**：shell 脚本被转成 CRLF 后在 Linux 执行报 `^M: bad interpreter`

**其他必知配置**：
```bash
core.ignorecase    # Windows/macOS 默认 true，导致只改文件名大小写 Git 不识别
core.filemode      # Windows 上应设 false，否则 NTFS 没有执行位会显示大量 mode change
core.quotepath false   # 让中文文件名正常显示而不是 \344\275\240
user.name / user.email
```

**大小写改名的正确姿势**：
```bash
git mv UserService.java userservice.java     # 正确
mv a.java A.java && git add -A               # 有时不生效
# 实在不行：core.ignorecase=false 之后再操作，或分两步：先改成 tmp 再改成目标名
```

**.gitignore 规则**：
- **只对未跟踪文件生效**。已提交的文件要先 `git rm --cached <file>` 才能被忽略
- `!` 是**否定规则**：`*.log` 后写 `!important.log` 表示不忽略它
- 以 `/` 开头只匹配仓库根；以 `/` 结尾只匹配目录；不含 `/` 的匹配任意层级的同名文件/目录
- 全局忽略：`git config --global core.excludesFile ~/.gitignore_global`（放 IDE 配置、.DS_Store）

### 2.10 协作工作流选型

| 模型 | 分支结构 | 适用 | 结论 |
|---|---|---|---|
| **GitFlow** | master + develop + feature/release/hotfix | 版本化发布的产品（客户端、嵌入式、SaaS 大版本） | 分支多、流程重，持续交付场景下太慢 |
| **GitHub Flow** | main + feature PR | 开源项目、SaaS | 简单，但要求每次合并都可发布 |
| **Trunk-Based** | 主干 + 短命分支（<1~2 天）或直接在主干 | **持续交付首选**，配合特性开关 | 需要较强的 CI 和自动化测试保障 |

**当前主流结论：持续交付优先 Trunk-Based + 短命特性分支 + PR 评审 + 特性开关（Feature Flag）。**

配套实践：
- 保护分支（禁止强推、要求 PR、要求 CI 通过、要求 N 个 reviewer approve）
- PR 模板（描述改动、影响面、测试情况、回滚方案）
- Conventional Commits（`feat:` `fix:` `docs:` `refactor:` `perf:` `test:` `chore:`）便于自动生成 CHANGELOG
- 提交粒度：一个 commit 做一件事，能独立编译、能通过测试

### 2.11 进阶

**submodule vs subtree**：

| | submodule | subtree |
|---|---|---|
| 本质 | 父仓库记录子仓库的 commit SHA（一个指针） | 把子仓库代码**复制进**父仓库 |
| clone | 需要 `--recursive` 或 `submodule update --init` | 一次性拿到全部 |
| 更新 | 子仓库独立演进，父仓库手动更新指针 | 需要 subtree pull/push 同步 |
| 适用 | 强依赖版本锁定、子项目独立开发 | 希望单仓库自包含、构建简单 |

```bash
git submodule add <url> libs/common
git submodule update --init --recursive
git clone --recursive <url>          # 一步到位
# 删除 submodule 必须手动清理四处：
#   1. git submodule deinit -f libs/common
#   2. git rm -f libs/common
#   3. 删除 .gitmodules 里的对应条目
#   4. rm -rf .git/modules/libs/common
```

**Git hooks**：位于 `.git/hooks/`，去掉 `.sample` 后缀即可生效。
- `pre-commit`：跑 lint、格式化、单测（推荐用 husky / pre-commit 框架统一管理）
- `commit-msg`：校验提交信息规范
- `pre-push`：跑完整测试
- **限制**：client hooks 不随 clone 分发（`.git` 不参与版本控制），团队要靠 `core.hooksPath` 指向仓库内目录 + 初始化脚本来统一

```bash
git config core.hooksPath .githooks   # 把 hooks 目录纳入版本控制
```

**提交签名**：
```bash
git commit -S -m "msg"        # GPG 签名
git log --show-signature      # 验证
# 部分开源项目要求 DCO：git commit -s  （Signed-off-by 行）
```

**大文件与历史清理**：
- Git LFS：`git lfs track "*.psd"` → 文件内容存 LFS 服务器，仓库里只存指针
- **清理历史中的大文件/敏感信息**：BFG Repo-Cleaner 或 `git filter-repo`
  ```bash
  git filter-repo --path secrets.yml --invert-paths    # 删除某文件的所有历史
  git filter-repo --strip-blobs-bigger-than 10M
  ```
  **警告：这会重写所有 commit SHA，必须通知全团队重新 clone，且需要强推所有分支和 tag。**

**超大仓库**：
```bash
git clone --depth 1 <url>                    # 浅克隆，只拿最新一次提交
git clone --filter=blob:none <url>           # partial clone，按需下载 blob
git sparse-checkout set src/main api         # 稀疏检出，只签出指定目录
```

## 三、高频踩坑

| 现象 | 根因 | 解法 |
|---|---|---|
| 整个文件显示为已修改，实际没动内容 | 换行符在 CRLF/LF 间被转换，或文件权限位变化 | 加 `.gitattributes` 统一 `eol=lf`；`core.filemode=false` |
| `git reset --hard` 后代码找不回来了 | 以为没救 | `git reflog` 找 SHA → `git reset --hard <sha>`；再不行 `git fsck --lost-found` |
| 强推把同事的提交冲掉了 | 用了 `git push --force` | 永远用 `--force-with-lease`；禁止对保护分支强推 |
| 对公共分支 rebase 后团队历史 diverge | rebase 改写了已推送的 commit SHA | 严禁对已推送的公共分支 rebase；只 rebase 本地未推送的提交 |
| 解决冲突时选错了方向 | `--ours`/`--theirs` 在 merge 和 rebase 下含义相反 | **merge 时** ours=当前分支、theirs=被合并分支；**rebase 时** ours=目标分支（上游）、theirs=你正在重放的提交。不确定就用编辑模式手动看 |
| rebase 中途乱了，越搞越糟 | 不知道有中止命令 | `git rebase --abort` 一键回到起点；单个冲突也可用 `git rebase --skip` / `--continue` |
| detached HEAD 下提交后 checkout 走了，代码丢了 | HEAD 直接指向 commit，没有分支引用它 | `git reflog` 找到 SHA → `git checkout -b save <sha>` |
| `.gitignore` 写了但不生效 | 只作用于未跟踪文件，已提交的文件不受影响 | `git rm --cached <file>` 后再提交；清理历史要用 filter-repo |
| `git pull` 后历史里一堆 `Merge branch 'main' of ...` | 每次 pull 都产生 merge commit | `git pull --rebase` 或 `git config --global pull.rebase true` |
| shell 脚本在 Linux 上执行报 `^M: bad interpreter` | 文件被转成了 CRLF | `.gitattributes` 里对 `*.sh` 指定 `eol=lf`；已提交的重新转换 |
| 只改文件名大小写，Git 不识别 | `core.ignorecase=true`（Windows/macOS 默认） | 用 `git mv` 改名；或临时设 `core.ignorecase=false`；或先改成中间名再改成目标名 |
| submodule 更新不到位导致构建不一致 | clone 时没加 `--recursive`，或子模块指针未更新 | `git submodule update --init --recursive`；CI 里必须显式执行 |
| `git add .` 把不该提交的文件加进去了 | 没有 .gitignore 或忽略了但文件已被跟踪 | 提交前 `git status` + `git diff --cached`；配 pre-commit 检查 |
| `git stash pop` 后冲突，改动找不到了 | stash 被 apply 成功但因冲突未自动删除，或 pop 失败 | `git stash list` 通常还在；用 `git stash branch` 从 stash 建分支解决冲突 |
| 仓库体积越来越大，clone 极慢 | 历史中提交过大文件 | `git filter-repo` 或 BFG 清理历史；新项目一开始就上 Git LFS |
| 误把密钥提交到公开仓库 | 提交即泄露（GitHub 有爬虫分钟级抓取） | **立刻吊销密钥**（改密钥优先于改历史）；再用 filter-repo 清历史 |
| `git cherry-pick` 后提交与原分支关联丢失 | cherry-pick 产生新 SHA | 加 `-x` 会在 message 里记录来源 commit，便于追溯 |

## 四、往下钻的入口

**源码级**：
- `.git/` 目录结构逐个看一遍：`HEAD`、`config`、`index`、`objects/`、`refs/`、`logs/`、`packed-refs`
- `git cat-file -p <sha>` 逐层展开 commit → tree → blob，亲手验证"快照"这件事
- `git hash-object -w test.txt` 手动生成一个 blob，验证内容寻址

**配置清单（建议一次性设好）**：
```bash
git config --global pull.rebase true
git config --global core.autocrlf input          # 或 Windows 上 true
git config --global core.quotepath false
git config --global core.editor "code --wait"
git config --global alias.lg "log --graph --oneline --decorate --all"
git config --global alias.st status
git config --global alias.unstage "restore --staged"
git config --global alias.undo "reset --soft HEAD~1"
git config --global rerere.enabled true          # 自动记录冲突解决方案，重复 rebase 时复用
```

**官方文档具体章节**：
- *Pro Git* 第 10 章 Git Internals（Plumbing and Porcelain、Git Objects、Packfiles、Refspec）—— 免费在线，讲底层最权威的一章
- `git help <command>` 的官方手册，`githelp revisions` 讲 A..B 与 A...B 的语法
- `git bisect` / `git rebase` 的官方手册里的图示

**进阶工具**：`git filter-repo`（官方推荐替代 filter-branch）、BFG Repo-Cleaner、`tig`（终端 UI）、`lazygit`（终端 UI）、`delta`（更好的 diff 渲染）

## 五、达标自检

1. 一次 `git commit` 在 `.git/objects` 里生成了哪几类对象？它们的引用关系是怎样的？
2. Git 存的是 diff 还是快照？为什么"全量快照"不会让仓库爆炸？
3. `HEAD`、分支、`refs/remotes/origin/xxx` 三者分别是什么？`git checkout <commit>` 之后 HEAD 变成什么状态？
4. `git reset --soft` / `--mixed` / `--hard` 分别移动了哪些区域？请分别给出一个适用场景。
5. 已经 push 到公共分支的错误提交，应该用 revert 还是 reset？为什么？
6. `git reset --hard` 后发现删错了，怎么救？reflog 存在哪里？保留多久？
7. reflog 也没了，还有什么办法找回丢失的 commit？
8. 为什么绝对不能在已推送的公共分支上 rebase？具体会发生什么？
9. `git diff A..B` 和 `git diff A...B` 有什么差别？看 PR 的改动应该用哪个？
10. merge 的 fast-forward 和 `--no-ff` 该怎么选？为什么有些团队强制要求 `--no-ff`？
11. merge 冲突时 `--ours` / `--theirs` 分别指谁？在 rebase 场景下还一样吗？
12. `rebase -i` 里的 squash 和 fixup 有什么区别？调整行顺序会引发什么问题？
13. 你误删了一个还没合并的分支，怎么找回来？
14. `git stash` 默认会保存未跟踪的文件吗？`git stash pop` 和 `apply` 应该优先用哪个，为什么？
15. `git push --force` 和 `--force-with-lease` 的区别是什么？后者的检查依据是什么？
16. 用 `git bisect` 定位一个 bug 引入点的完整命令流程是什么？如何自动化？
17. `core.autocrlf` 的 true / input / false 分别做了什么转换？为什么推荐用 `.gitattributes` 而不是靠本地配置？
18. `.gitignore` 对已跟踪的文件为什么不起作用？正确的处理步骤是什么？
19. Windows 上只改文件名大小写为什么 Git 不识别？怎么正确处理？
20. submodule 和 subtree 的本质区别是什么？删除一个 submodule 需要清理哪些地方？
21. 你要清理仓库历史里一个 200MB 的文件，应该怎么操作？这个操作会给团队带来什么后果？
22. 你看到同事的 PR 里有 30 个 commit，其中 5 个是"fix typo"。你会建议他怎么做？理由是什么？
