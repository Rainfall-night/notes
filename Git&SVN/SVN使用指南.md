# SVN 使用指南

> 基于 Windows + TortoiseSVN 1.14.9(内置 svn 1.14.5)环境,以 .NET 后端项目(如 `STWebsiteBackEnd_V2`)的实际场景为例。
> 练习仓库用本地 `file://` 协议,无需服务器。

---

## 一、SVN 是什么

SVN 是**集中式版本控制**:所有历史只存在**服务器仓库**里,你本地只有一个**工作副本**。

```
服务器仓库(Repository)          ← 唯一权威,历史都在这里
   │  svn checkout / svn update
   ▼
本地工作副本(Working Copy)      ← 你磁盘上的文件,含 .svn 隐藏文件夹
   │  svn commit
   ▼
(回到服务器)
```

- **版本号是全局递增整数**:`r1`、`r2`、`r123`,整个仓库统一编号
- **没有暂存区、没有本地提交**:改完要么 `svn commit`(上服务器),要么丢弃
- 本地练习无需服务器:用 `file:///盘符:/路径` 直接访问本地仓库

---

## 二、环境准备

1. 安装 **TortoiseSVN**(安装时勾选 **command line client tools**,否则没有命令行 `svn`)
2. 把安装目录的 `bin` 文件夹加进 **PATH**(如 `D:\Program Files\TorToiseSVN\bin`)
3. 重开终端验证:

```powershell
svn --version          # 应打印版本号和模块列表
svnadmin --version     # 应能出版本号(建仓工具)
```

---

## 三、新建仓库(比 Git 复杂,重点)

Git 一条 `git init` 就有仓库;SVN 要两步:**建仓库实体 + 建标准目录结构**。

### 3.1 建仓库实体(相当于 git init)

```powershell
svnadmin create D:\SVN\myrepo
```

- 在 `D:\SVN\myrepo` 生成一个**仓库目录**(里面有 `db/`、`conf/`、`hooks/` 等内部结构)
- 这是"服务器视角",**不能手工往里放文件**,只能通过 SVN 命令操作

### 3.2 建标准目录结构(trunk / branches / tags)

SVN 没有"分支"这个概念是内建的,它靠目录布局约定:

```
仓库根
├── trunk/      # 主干 ≈ 稳定版本,平时在这开发
├── branches/   # 功能/发布分支
└── tags/       # 发布标签(只读快照)
```

**推荐用 `svn mkdir` 一步建好**(简单,不需要临时目录):

```powershell
svn mkdir file:///D:/SVN/myrepo/trunk file:///D:/SVN/myrepo/branches file:///D:/SVN/myrepo/tags -m "初始化标准目录结构"
svn list file:///D:/SVN/myrepo        # 验证:应看到 trunk、branches、tags
```

> 另一种方式 `svn import` 也常见——适合**首次把已写好的项目一次性导入仓库**:
> ```powershell
> svn import D:\MyProject file:///D:/SVN/myrepo/trunk -m "首次导入项目"
> ```

### 3.3 想用"真正的服务器地址"?启动 svnserve(可选)

```powershell
svnserve -d -r D:\SVN        # -d 后台,-r 指定根目录,监听 3690 端口
svn checkout svn://localhost/myrepo   # 用 svn:// 地址签出
taskkill /im svnserve.exe    # 用完关闭
```

### 3.4 对比 Git

| | Git | SVN |
|---|---|---|
| 建仓库 | `git init` | `svnadmin create` + `svn mkdir`(建标准目录) |
| 远程地址 | `git remote add origin url` | 建仓时就定了(`file://` 或 `svn://` 或 `http://`) |
| 标准结构 | 没有硬性约定 | **trunk/branches/tags 是约定俗成** |

---

## 四、拿到代码:`svn checkout`(对应 git clone)

```powershell
svn checkout <仓库URL> [本地目录]
svn checkout file:///D:/SVN/myrepo/trunk D:\SVN\workcopy
```

- 签出后生成**工作副本**,里面带 `.svn` 隐藏文件夹
- **注意:签出的是 `.../trunk`**,因为仓库根目录下还有 branches/tags

---

## 五、每天开工:`svn update`(对应 git pull)

```powershell
svn update          # 把服务器新提交合并进工作副本
svn update -r 120   # 临时把工作副本切到 r120 看旧代码
```

- SVN 是集中式,同事随时可能提交,**开工前必 update**,越晚更新冲突越多
- update 是"合并",不是覆盖——改到同一行会冲突(见第十二章)

---

## 六、改代码:看状态与差异

```powershell
svn status          # 看哪些文件变了(简写 -q)
svn diff            # 看本地改动内容
svn diff -r 120:125 # 对比两个历史版本
```

`svn status` 输出第一列的状态标记:

| 标记 | 含义 |
|---|---|
| `?` | 新文件,未纳入版本控制 |
| `A` | 已 add,待提交 |
| `M` | 已修改,待提交 |
| `D` | 已删除 |
| `C` | 冲突 |
| `!` | 文件缺失 |

---

## 七、新文件:**先 `svn add`(最易踩坑)**

```powershell
svn add 新文件.txt       # 把新文件纳入版本控制
svn add .                # 添加当前目录所有未跟踪的新文件
```

> ⚠️ **SVN 里"新文件"必须 `svn add` 才会被提交**——这是新手第一坑:忘了 add,commit 完发现文件根本没进仓库。
> Git 的 `git add` 是"暂存改动";SVN 的 `svn add` 是"把新文件纳入版本控制",改了旧文件**不需要** add,直接 commit。

---

## 八、提交:`svn commit`(对应 git push+commit 一步到位)

```powershell
svn commit -m "修复登录bug"        # 提交所有改动到服务器
svn commit 某个文件 -m "只提交这个"  # 只提交指定文件
```

- **SVN 没有 push**——commit 就是直接上服务器,必须联网
- 提交后会得到一个**新的全局版本号**,如 `r2`
- 提交信息 `-m` 必填,写清楚改了什么,方便团队看历史

---

## 九、看历史与对比

```powershell
svn log                # 提交历史(r2 修复登录bug / r1 初始化)
svn log -l 5           # 最近 5 条
svn log -v             # 显示每次提交改了哪些文件
svn log --search "bug" # 按信息搜索
svn log 文件.txt        # 看某个文件的历史
svn blame 文件.cs       # 逐行显示谁、哪个版本改的
svn diff -r 120:125    # 对比两个版本差异
```

> 版本号是数字,比 Git 的哈希好记:"回退到 r120"一句话就懂。

---

## 十、撤销与回退

### 撤销"未提交"的改动(不可恢复!)

```powershell
svn revert 文件.txt      # 丢弃该文件未提交的修改
svn revert -R .         # 递归撤销整个目录
```
> ⚠️ `svn revert` 不可恢复(SVN 没有 reflog),想清楚再执行。

### 回退"已提交"的历史(只能反向合并,不能抹)

SVN 历史在服务器上改不了,回退 = **基于旧版本再提交一次**,历史留两条记录:

```powershell
svn merge -r 125:120 .     # 算出 r120→r125 的反向改动,应用到工作副本(内容变回 r120)
svn commit -m "回退r125的改动"   # 提交成新版本 r126
```

### 临时看旧代码

```powershell
svn update -r 120       # 工作副本整个回到 r120(临时看)
svn cat -r 120 文件.txt  # 直接输出 r120 的某文件内容,不改变工作副本
```

---

## 十一、分支与标签

### 概念

SVN 的分支**不是指针,是目录拷贝**:

```
file:///D:/SVN/myrepo/
├── trunk/            # 主干
├── branches/dev      # 分支 = trunk 复制一份
└── tags/v1.0         # 标签 = 只读快照
```

### 常用操作

```powershell
# 建分支(复制目录)
svn copy file:///D:/SVN/myrepo/trunk file:///D:/SVN/myrepo/branches/dev -m "创建dev分支"

# 切到分支 / 切回主干
svn switch file:///D:/SVN/myrepo/branches/dev
svn switch file:///D:/SVN/myrepo/trunk

# 在分支上提交
svn commit -m "dev分支的改动"

# 把分支合并进主干
svn switch file:///D:/SVN/myrepo/trunk     # 先回主干
svn merge file:///D:/SVN/myrepo/branches/dev
svn commit -m "合并dev到主干"

# 删分支
svn delete file:///D:/SVN/myrepo/branches/dev -m "删除已合并分支"
```

> 💡 SVN 里 **update 是"把服务器合并到自己这边"**,**merge 是"把别的 URL 的差异合并进来"**,别混。
> 分支是目录拷贝、成本高,**小改动直接在主干做**,别轻易开分支。

---

## 十二、冲突解决

### 什么时候冲突

你和同事改了**同一文件同一行**,`svn update` 或 `svn merge` 时不知道用谁的 → 冲突。

### 冲突时生成三个文件(保险设计)

| 文件 | 含义 |
|---|---|
| `file.cs` | 冲突本体(内部有 `<<<<<<< .mine` 标记) |
| `file.cs.mine` | 你的版本 |
| `file.cs.rOLD` | 基础版本(共同起点) |
| `file.cs.rNEW` | 服务器最新版本(同事的) |

冲突标记长这样:

```
<<<<<<< .mine
你的代码
=======
>>>>>>> .rNEW
同事的代码
```

### 解决流程

```powershell
svn update                          # ① 更新,报冲突
# ② 解决:手动编辑删标记 / TortoiseSVN 右键 Edit conflicts / svn resolve --accept
svn resolve 文件.cs                  # ③ 标记已解决(必须!)
svn commit -m "解决冲突"              # ④ 提交
```

> ⚠️ **必须 `svn resolve` 标记已解决**,否则该文件无法 commit。
> TortoiseSVN 的图形合并(右键 → Edit conflicts)是三栏对比,新手强烈推荐用它。

---

## 十三、忽略文件(对应 .gitignore)

Git 用 `.gitignore` 文件;SVN 用目录的 **`svn:ignore` 属性**:

```powershell
# 在项目根目录设置忽略(注意:SVN 属性是换行分隔的)
svn propset svn:ignore "bin
obj
*.user
.vs" .

# 或用编辑器交互式输入
svn propedit svn:ignore .

# 查看 / 校验
svn propget svn:ignore .
```

> 从 Git 项目迁到 SVN,要把 `.gitignore` 的规则转成 `svn:ignore` 属性。

---

## 十四、实际场景速查表

| 你想干什么 | 用哪条命令 |
|---|---|
| 第一次拿到项目 | `svn checkout <url>/trunk` |
| 每天开工拉最新 | `svn update` |
| 看改了哪些文件 | `svn status` |
| 看改了什么内容 | `svn diff` |
| 新建了文件要提交 | **`svn add 文件`** → `svn commit -m "说明"` |
| 改完提交 | `svn commit -m "说明"`(不用 add) |
| 看提交历史 | `svn log` / `svn log -l 10` |
| 看某行谁改的 | `svn blame 文件` |
| 改错了想丢弃 | `svn revert 文件`(不可恢复!) |
| 想回退到旧版本 | `svn merge -r 新:旧 .` → `svn commit` |
| 开个分支干活 | `svn copy <url>/trunk <url>/branches/xxx -m "建分支"` → `svn switch` |
| 把分支合并回主干 | `svn switch <url>/trunk` → `svn merge <url>/branches/xxx` → `svn commit` |
| 更新时冲突了 | 手动/TortoiseSVN 解决 → `svn resolve 文件` → `svn commit` |
| 发布打标签 | `svn copy <url>/trunk <url>/tags/v1.0 -m "发布"` |
| 忽略 bin/obj 等 | `svn propedit svn:ignore .` |

---

## 附:SVN 完整开发循环(背下来)

```powershell
# ① 拿到代码
svn checkout file:///D:/SVN/myrepo/trunk D:\work

# ② 每天开工
cd D:\work
svn update

# ③ 改代码...然后看状态
svn status
svn diff

# ④ 新文件要 add
svn add 新文件

# ⑤ 提交
svn commit -m "说明"
```
