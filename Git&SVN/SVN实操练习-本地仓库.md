# SVN 实操练习 · 本地仓库版

> **仓库位置**:`file:///E:/workSpace/myrepo`
> **环境**:Win11 终端(可直接识别 `svn` 命令;若在 VS Code 终端使用,需先 `$env:Path += ";D:\Program Files\TorToiseSVN\bin"` 或配好 PATH)
> 本手册所有命令都按你的实际仓库路径写好,照着抄即可。

---

## 前置:确认仓库可访问

```powershell
svn list file:///E:/workSpace/myrepo
```

- 能执行(空仓库会没输出或报"未找到路径")→ 仓库路径没问题,继续;
- 报 `Illegal repository URL` → 路径写错,检查 `file:///E:/workSpace/myrepo` 的写法(三个斜杠 + 盘符 + 正斜杠)。

---

## ① 建标准目录结构(trunk / branches / tags)

```powershell
# 在仓库根下创建三个标准目录
svn mkdir file:///E:/workSpace/myrepo/trunk file:///E:/workSpace/myrepo/branches file:///E:/workSpace/myrepo/tags -m "初始化标准目录结构"

# 验证:应输出 trunk、branches、tags
svn list file:///E:/workSpace/myrepo
```

> 这是 SVN 约定俗成的布局:trunk=主干、branches=分支、tags=标签。`svn mkdir` 是直接在仓库里建目录,不需要本地临时文件夹。

---

## ② 签出工作副本(对应 git clone)

```powershell
# 注意:签出的是 .../trunk,不是仓库根
svn checkout file:///E:/workSpace/myrepo/trunk E:\SVN\workcopy

# 进入工作副本(以后干活的地方)
cd E:\SVN\workcopy
```

> 签出后 `workcopy` 里会多一个隐藏的 `.svn` 文件夹,这就是"工作副本"的标志。

---

## ③ 第一个提交(新文件必须 svn add!)

```powershell
# 建一个新文件,写点内容(比如 hello svn)
notepad hello.txt

# 看状态:应显示  ?  hello.txt(? = 未纳入版本控制)
svn status

# ⚠️ 新文件必须先 svn add,否则 commit 不会带上它
svn add hello.txt

# 提交
svn commit -m "第一个提交"

# 看历史:应看到 r1
svn log
```

---

## ④ 修改 + 提交(旧文件不用 add,直接 commit)

```powershell
# 改内容(比如 hello svn v2)
notepad hello.txt

# 应显示  M  hello.txt(已修改)
svn status

# 看具体改了什么
svn diff

# 旧文件直接提交
svn commit -m "第二个提交"

# 看每次提交改了哪些文件
svn log -v
```

---

## ⑤ 分支练习(重点)

```powershell
# 1. 建分支 = 把 trunk 复制一份到 branches/dev
svn copy file:///E:/workSpace/myrepo/trunk file:///E:/workSpace/myrepo/branches/dev -m "创建dev分支"

# 2. 工作副本切到 dev 分支(对应 git checkout)
svn switch file:///E:/workSpace/myrepo/branches/dev

# 3. 在 dev 上改点东西并提交
notepad hello.txt
svn commit -m "dev分支上的修改"

# 4. 切回主干
svn switch file:///E:/workSpace/myrepo/trunk

# 5. 把 dev 的改动合并进主干(对应 git merge)
svn merge file:///E:/workSpace/myrepo/branches/dev

# 6. 提交合并结果
svn commit -m "合并dev到主干"

# 7.(可选)删除已合并的分支
svn delete file:///E:/workSpace/myrepo/branches/dev -m "删除已合并分支"
```

> 容易混:SVN 里 **update 是"把服务器合并到自己这边"**,**merge 是"把别的 URL 的差异合并进来"**。

---

## ⑥ 冲突模拟(进阶,体验一下)

开**两个终端窗口**,各签出一份工作副本,改同一行再都提交:

```powershell
# 窗口 1
svn checkout file:///E:/workSpace/myrepo/trunk E:\SVN\wc1
notepad E:\SVN\wc1\hello.txt      # 把内容改成 A
svn commit -m "wc1 改 A"          # 先提交 → 成功

# 窗口 2(先不 update,直接改同一行)
svn checkout file:///E:/workSpace/myrepo/trunk E:\SVN\wc2
notepad E:\SVN\wc2\hello.txt      # 把同一行改成 B
svn commit -m "wc2 改 B"          # 报错:out of date / 冲突

# 解决:窗口 2 先 update → 手动改冲突标记 → resolve → commit
svn update
# 冲突文件旁会生成 hello.txt.mine / .r1 / .r2
# 手动编辑 hello.txt,删掉 <<<<<<< .mine ... ======= ... >>>>>>> 标记,保留想要的
svn resolve hello.txt
svn commit -m "解决冲突"
```

> 提示:冲突的图形化解法用 TortoiseSVN 右键 → Edit conflicts,三栏对比,新手推荐。

---

## 附:建仓后常用命令速查

| 你想干什么 | 命令 |
|---|---|
| 确认仓库 | `svn list file:///E:/workSpace/myrepo` |
| 签出工作副本 | `svn checkout file:///E:/workSpace/myrepo/trunk E:\SVN\workcopy` |
| 开工前拉最新 | `svn update` |
| 看状态 | `svn status` / `svn status -q` |
| 看改动内容 | `svn diff` |
| 新文件纳入版本 | **`svn add 文件`** |
| 提交 | `svn commit -m "说明"` |
| 看历史 | `svn log` / `svn log -v` / `svn log -l 5` |
| 看某行谁改的 | `svn blame 文件` |
| 建分支 | `svn copy <仓库>/trunk <仓库>/branches/xxx -m "说明"` |
| 切分支 | `svn switch <仓库>/branches/xxx` |
| 合并分支 | `svn switch <仓库>/trunk` → `svn merge <仓库>/branches/xxx` → `svn commit` |
| 丢弃未提交改动 | `svn revert 文件`(⚠️ 不可恢复) |
| 回退历史 | `svn merge -r 新:旧 .` → `svn commit` |
| 打标签 | `svn copy <仓库>/trunk <仓库>/tags/v1.0 -m "发布"` |
