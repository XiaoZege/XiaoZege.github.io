---
title: "批量改名为什么得先看一眼：file-tool 第二周"
date: 2026-11-01
draft: false
tags: ["Python", "CLI", "pytest"]
categories: ["项目记录"]
ShowToc: true
---

上周把 `rename` 命令写完了，一次都没敢在真文件夹上用过。它能批量改文件名，也能批量把文件名改错。

这周只干一件事：给它加个演习模式。先把要改什么列出来看一遍，确认没问题，再去掉开关真改。

## 我怕的是什么

写错一个字母是小事，后果不是。`old` 参数传错，可能把文件名里本来不该动的字符全替换掉。改错之后还能照着记录一个个改回去，只是麻烦。

真正救不回来的是文件被覆盖。如果两个文件改完之后撞上同一个名字，内容就被盖掉了，那份东西不会再回来。

所以这类工具都该有个只看不做的版本。这不是我发明的规矩，是批量操作的标准做法。

顺带说一句，我这版代码其实已经防了撞名：改之前先检查目标名是否存在，存在就跳过并打一行提示。但"参数写错"这种防不住，还是得靠人眼看一遍。

## 加一个 `--dry-run`

三处改动。

第一处，给函数加个开关：

```python
def rename(
    directory: str,
    old: str,
    new: str,
    dry_run: bool = typer.Option(False, "--dry-run", help="只预览，不真的改名"),
):
```

`bool` 是只管开和关的类型，默认 `False`。命令行里写了 `--dry-run` 它就变成 `True`，不写就一直是关的。

第二处，在循环里、真正改名那一句**前面**插一段：

```python
if dry_run:
    typer.echo(f"{item.name} -> {new_name}（预览，没动文件）")
    changed += 1
    continue
```

`continue` 是这里的关键。它的作用是跳过本轮剩下的代码，直接进入下一个文件。因为这段写在 `item.rename(...)` 前面，所以演习时那句真改名永远执行不到。

我一开始以为"只看不改"得用 if 把改名那句包起来，写到这里才发现不用。位置放对了，`continue` 就把活干了，一行判断解决。

第三处，把结尾提示从"共改名"改成"共处理"。演习时一个文件都没改名，还写"共改名"就是骗人。

## 跑一下

不敢在真文件夹上试，先造三个空文件：

```powershell
mkdir C:\dev\tmp-rename -Force
New-Item C:\dev\tmp-rename\old1.txt, C:\dev\tmp-rename\old2.txt, C:\dev\tmp-rename\old3.txt -ItemType File
```

先演习：

```bash
uv run file-tool rename C:\dev\tmp-rename old new --dry-run
```

```
old1.txt -> new1.txt（预览，没动文件）
old2.txt -> new2.txt（预览，没动文件）
old3.txt -> new3.txt（预览，没动文件）
共处理 3 个文件
```

然后 `dir` 一下，文件名还是 `old` 开头。确实没动。

去掉 `--dry-run` 再跑，同样的三行会出来，只是没有"预览"两个字。这时候 `dir`，才真的变成 `new1.txt` 那几个。

这个顺序我建议固定下来：先演习、看输出、再真跑。多花十秒，省掉一次灾难。

## 关于计数器放哪

有件事我想过：`changed += 1` 到底该放哪。

演习那支我放在打印后面，真改那支放在 `rename` 后面。看起来有点重复，但我没有合并成一处。

先说实话：合不合并，跑出来的数字是一样的。真改那支如果 `rename` 失败会抛异常，命令当场中断，最后那句"共处理 X 个文件"根本打不出来。所以不存在"数错了还照样输出"。

我还是分成两支写，纯粹为了读起来清楚：演习干的事和真改干的事是两件不同的事，各写各的，以后改其中一支不用回头看另一支。

但这个位置以后会变重要。哪天我给它加个 try/except，说"某个文件改不了就跳过，接着处理下一个"，那计数放哪就真的影响结果了——失败也照样 +1 的话，最后报出来的数就是假的。

## 顺手撞了一下测试

改完措辞，跑 `uv run pytest`，红了一个。

失败信息里真正有用的只有两行，一个 `>` 一个 `E`：

```
>       assert "共改名 2 个文件" in result.stdout
E       AssertionError: assert '共改名 2 个文件' in '...共处理 2 个文件\n'
```

`>` 指出哪一行断了，`E` 告诉你为什么。翻译过来就是：我拿着"共改名"去程序输出里找，没找着，程序现在说的是"共处理"。

这不算事故。测试的作用就是把我和程序之间的约定写死，约定改了它立刻响。把期望值改成"共处理"就过了。

再补一条专门测演习模式的：

```python
def test_rename_dry_run_does_not_touch_files(tmp_path):
    (tmp_path / "old1.txt").write_text("")
    (tmp_path / "old2.txt").write_text("")

    result = runner.invoke(app, ["rename", str(tmp_path), "old", "new", "--dry-run"])

    assert result.exit_code == 0
    assert (tmp_path / "old1.txt").exists()
    assert not (tmp_path / "new1.txt").exists()
    assert "预览" in result.stdout
```

重点是那句 `assert not (tmp_path / "new1.txt").exists()`，断言新文件名**不存在**。

写测试容易只顾着测"该发生的发生了"，忘了另一半"不该发生的没发生"。演习模式的全部承诺就是不动文件，这一条不测，等于没测。

## 这周的收尾

十条测试全过，ruff 没意见，推上去 CI 也是绿的。

`stats` 也顺手加了个 `--ext`，只统计指定后缀，比如 `uv run file-tool stats . --ext .py`。加上之后数字从 17 掉到 6，看着舒服。

下周想给 `dedupe` 也补个演习。它比 `rename` 危险——它直接重写 CSV 原文件，跑错了原表就没了。
