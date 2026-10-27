---
title: "用 uv 起一个 Python CLI 项目：file-tool 第一周"
date: 2026-10-27
draft: false
tags: ["Python", "uv", "pytest"]
categories: ["项目记录"]
ShowToc: true
---

这周的目标很小：写一个能在命令行里跑的小工具，顺手把 uv、pytest、ruff、GitHub Actions 摸一遍。

工具就三个命令：按扩展名统计文件数、批量改文件名、CSV 去重。选这个是因为我真的需要——截图、导出的表格、临时脚本堆在文件夹里，改名全靠手点。

## 建项目

四条命令，没装 pip，也没手动建虚拟环境。

```bash
uv init file-tool
cd file-tool
uv add typer
uv add --dev pytest ruff
```

`uv init` 生成的东西里，`pyproject.toml` 记依赖和项目元信息，`.python-version` 里就一个数字 `3.13`，`.gitignore` 已经帮你把 `.venv` 排除了。

`uv add` 做两件事：依赖写进 `pyproject.toml`，确定版本写进 `uv.lock`。锁文件比前面那个重要。没有它，半年后别人装这个项目，装到的是"当前最新的 typer"，我这边跑通的代码在他那边可能直接报错。有它，装到的就是和我一模一样的一套。

## 第一个坑：.venv 被算进去了

第一版写完，我在项目根目录跑：

```bash
uv run file-tool stats .
```

输出"共 1153 个文件"。而项目里我自己写的文件加起来不到十个。

原因是 `.venv`。虚拟环境里装着 typer、pytest、ruff 和它们依赖的一切，几千个文件全被 `rglob("*")` 一起数进去了。

修法是加一个跳过名单：

```python
SKIP_DIRS = {".git", ".venv", "__pycache__", "node_modules", ".idea"}
```

再判断相对路径的每一段：

```python
if any(part in SKIP_DIRS for part in item.relative_to(root).parts):
    continue
```

`relative_to(root)` 把绝对路径裁成相对路径，`.parts` 再拆成一段一段，只要有一段命中名单就跳过。改完再跑，17 个。

## 第二个坑：测试断言一直失败

`exit_code` 是对的，`result.stdout` 里就是找不到那句话。

```python
result = runner.invoke(app, ["stats", str(tmp_path / "not-exist")])
assert result.exit_code == 1
assert "不是目录" in result.stdout
```

查了一会儿才反应过来。命令里我写的是：

```python
typer.echo(f"不是目录：{root}", err=True)
```

`err=True` 把这行送到 stderr 了，而 `result.stdout` 只收 stdout。

记下这个不是因为它难，是因为我第一次意识到 stdout 和 stderr 是两个通道。以前在终端里看报错，红字黑字混在一起，根本没想过它们走的路不一样。

## 三个命令最后长什么样

`rename` 的核心就四行：

```python
for item in root.iterdir():
    new_name = item.name.replace(old, new)
    if new_name == item.name:
        continue
    item.rename(root / new_name)
```

`iterdir()` 只列当前这一层，不往子目录钻。

`if new_name == item.name` 那行不能省——old 没出现在文件名里的时候，`replace` 会原封不动返回原名，这时候再 rename 一次没有意义。

还发现一个容易忽略的问题：目标文件名已存在时，`rename` 会直接覆盖。所以加了一条判断，同名就跳过并打印一行提示。

`dedupe` 用 `csv.DictReader` 读，每行变成 dict，把指定列的值塞进 `set`，见过就丢掉。

## CI

`.github/workflows/ci.yaml` 二十行不到：

```yaml
- uses: actions/checkout@v4
- uses: astral-sh/setup-uv@v5
- run: uv sync --dev
- run: uv run ruff check .
- run: uv run pytest
```

`setup-uv` 会顺手配好缓存，第二次跑会快不少。

## 还欠着的东西

- `rename` 不递归，子目录里的文件改不到
- `dedupe` 只能按一列去重，多列组合的情况写不出来
- 没有 `--dry-run`，改名之前不能先预览一遍

第三个最要紧。批量改名一旦改错，靠手改回来比重写代码还累。

---

代码在 [XiaoZege/file-tool](https://github.com/XiaoZege/file-tool)，9 个测试本地全绿。
