# 交接：在 Windows 上继续 windows-compat 分支

写于 2026-11-03，Mac 上。这份文件给两方看：一是到 Windows 电脑上的我自己，二是在 Windows 上接手的 Claude Code。
Claude 接手时先读完这份文件，再按「交给 Claude 的待办」继续往下做。

## 目标

让这个 fork（`Jeremy-creater/mdict-index`）在原生 Windows 上完整可用：检索网站、统一词典库、词汇清单、平行语料、文本分析器这五部分都能跑。
`main` 分支保持和原作者 `xiaolai` 一致，方便每月同步；所有兼容改动都放在 `windows-compat` 分支。

## 已经做完的（Mac 上）

| 改动 | 文件 | 原因 |
|---|---|---|
| 给 138 处文本读写补上 `encoding="utf-8"` | `scripts/`、`inventories/`、`parallel_corpus/`、`analyzer/` 和各处 `tests/` 下的 29 个文件 | Windows 默认按系统区域编码（中文系统是 GBK）读写文件，读到 UTF-8 的中文会报错或乱码 |
| `fcntl.flock` 改成跨平台的 `hold()`，Windows 上用 `msvcrt.locking` | `parallel_corpus/parallel/extract.py` | `fcntl` 只在 Mac 和 Linux 上有，原来的代码在 Windows 上一导入就失败 |
| `PYTHONPATH` 的拼接改成 `os.pathsep.join(...)` | `inventories/build.py`、`parallel_corpus/build.py` | 原来写死成 `scripts:inventories`，而 Windows 的路径分隔符是 `;` |
| 新增 `.gitattributes`（`* text=auto eol=lf`） | 仓库根目录 | 避免 Windows 上的 Git 把换行符转成 CRLF，导致按字节比对的测试失败 |
| 新增 Windows 云端测试流程 | `.github/workflows/windows.yml` | 推送到这个分支时，GitHub 会在 Windows 机器上跑全部测试，再构建一次网站数据并和已提交的比对 |
| README 新增 Windows 一节 | `README.md` | 说明 Windows 上命令写法的三处不同 |

在 Mac 上的验证结果：
- 五套测试全部通过。主测试 557 个（跳过 1 个），平行语料 219 个，词汇清单 272 个，文本分析器 29 个，网页（Node）33 个。
- 开启 `-X warn_default_encoding -W error::EncodingWarning` 后仍然全部通过，说明测试跑到的代码里，已经没有不指定编码的读写。
- 把区域编码设成 `LC_ALL=en_US.ISO8859-1` 来模拟非 UTF-8 系统，全部通过；修复之前，平行语料有 35 个测试失败。
- 在 Latin-1 编码下运行 `build_site.py`，生成的 `site/data/dicts.json` 和原版逐字节一致。

还没验证的：
- 在真正的 Windows 上运行。
- Windows 上的 `msvcrt` 锁分支：Mac 上执行不到这段代码。
- 完整的词典构建：要下载约 21 GB，按我的要求暂时不下载。

## 在 Windows 上的操作步骤

### 第 1 步：安装软件

| 软件 | 下载 | 注意 |
|---|---|---|
| Python 3.12 或更新 | <https://www.python.org/downloads/> | 安装第一屏勾选「Add python.exe to PATH」 |
| Git | <https://git-scm.com/download/win> | 一路默认 |
| Node.js（可选） | <https://nodejs.org/> | 只有跑网页测试才需要 |
| Claude Code | <https://claude.com/claude-code> | 用来在 Windows 上接着让 Claude 干活 |

Windows 10 和 11 自带 curl，下载词典要用到它，不用另外装。

### 第 2 步：把代码拉到本地

打开「开始」菜单，搜索 PowerShell 并打开，然后运行下面的命令。项目路径尽量短，放在空间大的盘上：以后完整构建最多要占 78 GB。

```powershell
cd D:\
git clone https://github.com/Jeremy-creater/mdict-index.git
cd mdict-index
git switch windows-compat
git log --oneline -3
```

`git log` 显示的第一行应该是「Make the tools run on Windows」那次提交。
如果 Git 弹出登录 GitHub 的窗口，用浏览器登录即可。

### 第 3 步：建虚拟环境，装依赖

```powershell
python -m venv .venv
.venv\Scripts\python.exe -m pip install -r requirements-dev.txt
.venv\Scripts\python.exe scripts\doctor.py
```

doctor 输出的最后一行应该是 `ready`。看到 `jev not installed` 可以忽略，不需要它。

### 第 4 步：跑测试

```powershell
$py = ".venv\Scripts\python.exe"
$env:PYTHONPATH = "scripts";                 & $py -m unittest discover -s tests
$env:PYTHONPATH = "scripts;parallel_corpus"; & $py -m unittest discover -s parallel_corpus/tests -t parallel_corpus/tests
$env:PYTHONPATH = "scripts;inventories";     & $py -m unittest discover -s inventories/tests -t inventories/tests
$env:PYTHONPATH = "scripts;analyzer";        & $py -m unittest discover -s analyzer/tests -t analyzer/tests
node --test (Get-ChildItem tests\*.test.mjs).FullName
```

每一套测试最后都应该显示 `OK`（Node 测试显示 `fail 0`）。

### 第 5 步：打开检索网站

```powershell
python -m http.server -d site 8000
```

在浏览器打开 <http://localhost:8000>，搜一下「牛津高阶」，能搜出结果就说明正常。按 `Ctrl + C` 停止。

### 第 6 步：让 Claude 接着做

在项目文件夹里启动 Claude Code：

```powershell
claude
```

然后对它说：

> 读 RESUME.md，按「交给 Claude 的待办」继续。

## 交给 Claude 的待办

- [ ] 在 Windows 上跑通第 3 步和第 4 步。有失败的话，找到原因后在 `windows-compat` 分支上修复，修完重跑全部五套测试。常见的问题有：用 `/` 拼接的路径字符串和 `str(Path)` 比较；Windows 上文件被打开时不能删除或替换，可能在 `os.replace`、`unlink` 时报错；`subprocess` 调用的程序找不到。
- [ ] 在 GitHub 网页上看 `.github/workflows/windows.yml` 有没有跑、是不是绿的。fork 的 Actions 默认可能是关的，要在仓库的 Actions 页面手动打开。
- [ ] 运行 `python scripts\build_site.py`，确认 `site/data/dicts.json` 和已提交的版本没有差异：`git diff --stat site/data`。这个脚本会把 `site/data/meta.json` 里的 `crawled` 改成当天日期，比对完用 `git checkout site/data/meta.json` 还原，不要提交。
- [ ] 不要下载词典，等我明确说要下载再做。到时按 README 的「From zero」一节操作，Windows 上的命令写法见下表。下载和构建中途断了，重新运行同一条命令就会续上。
- [ ] 全部验证通过后问我：要不要把 `windows-compat` 合并进 `main`，或者给原作者提 PR。在我回答之前，不要改动 `main`。

## Mac 和 Windows 的命令对照

| Mac 和 Linux | Windows PowerShell |
|---|---|
| `.venv/bin/python` | `.venv\Scripts\python.exe` |
| `PYTHONPATH=scripts 命令` | `$env:PYTHONPATH = "scripts"`，换行后再运行命令 |
| `PYTHONPATH=scripts:inventories` | `$env:PYTHONPATH = "scripts;inventories"`（`:` 换成 `;`） |
| `python3` | `python` |

## 相关文件

- 小白版使用指南（Obsidian，只在 Mac 上有）：`~/Sync/02_Obsidian/SecondBrain/200.Courses/Xiaolai/Tools/mdict-index-使用指南.md`。第五部分讲的是 Windows。
- 推荐的 25 部词典：`DICTIONARIES.md`。
