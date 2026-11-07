---
title: "装饰器到底解决什么问题：从重试说起"
date: 2026-11-07
draft: false
tags: ["Python", "装饰器", "异常处理"]
categories: ["项目记录"]
ShowToc: true
---

这周的任务是写三个装饰器。动手之前我一直觉得这事跟我没关系，装饰器不就是给函数套个壳，语法糖而已，用不着自己写。

真正写起来才发现，它是被一个特别具体的麻烦逼出来的。

## 麻烦本身

我在写一个小工具，其中一步要请求网页。第一次跑得好好的，第二次对方服务器没响应，程序直接红着退出。

但这不是我代码写错了，是网络偶尔抖一下。这种错误，等半秒再试一次，八成就能过。

问题是这段"再试一次"的逻辑得写在每个会失败的地方。写第一处的时候还好：

```python
try:
    resp = requests.get(url, timeout=3)
except Exception:
    time.sleep(0.5)
    resp = requests.get(url, timeout=3)
```

写到第二处、第三处，我就开始复制粘贴了。后面改起来也烦，逻辑散在七八个地方，改一次忘一处。

## 装饰器站的位置

这时候才明白装饰器是干什么的：它把"重试"这段跟业务无关的逻辑从函数里搬出去。

```python
@retry(times=3, delay=0.5)
def fetch(url):
    return requests.get(url, timeout=3)
```

`fetch` 里一行重试都没有，只干它自己该干的事。

顺序上有一点要认清楚：`@retry(...)` 不是"运行时给函数加个功能"，而是**在函数定义完的那一刻**，把它包成另一个东西。`fetch` 这个名字，从定义结束起指的就是包装后的版本，原函数已经被关进去了。

写成等价的普通代码更直白：

```python
fetch = retry(times=3, delay=0.5)(fetch)
```

## 为什么要三层

我一开始写了两层：

```python
def retry(func, times=3):
    ...
```

然后试着写 `@retry(times=3)`，直接报错。因为括号先算，`times=3` 传进去的时候 `func` 还是空的，后面要调 `func(...)` 的地方就崩了。

所以最外面那层只能收参数，收完吐出一个"还没接函数"的东西：

```python
def retry(times=3, delay=0.5):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(1, times + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    print(f"第 {attempt} 次失败：{e}")
                    if attempt == times:
                        raise
                    time.sleep(delay)
        return wrapper
    return decorator
```

`decorator` 和 `wrapper` 这两个名字不是我随便起的。别人写的库、我能翻到的源码，基本都这么叫。认名字比读实现快得多。

## 四个坑

### 一、错误不能被吞

一开始我写的是"失败就打印一句，然后接着往下走"。跑起来看着正常，其实外面完全不知道出过事，还以为拿到了数据。

`retry` 的规矩是：次数用完了，**把最后一个错误原样扔出去**。

```python
if attempt == times:
    raise
```

单独一个 `raise`，是把刚才抓到的那个异常重新扔一遍，不是造一个新的。报错位置还指在真出事的 `func` 里面，方便查。

### 二、别写光秃秃的 `except:`

我原本觉得 `except Exception` 和 `except:` 差不多，反正都是"什么错都抓"。查了一下才发现不是。

`KeyboardInterrupt`（就是我按 Ctrl+C）不归 `Exception` 管，它挂在 `BaseException` 下面。裸 `except:` 会把它也抓住，然后拿去重试。结果就是我在那儿猛按中断键，程序还在那儿睡半秒、再试一次。

按 Ctrl+C 是我想让它停，不是程序出错了。这俩不该混在一起。

### 三、加了装饰器，函数名字会变

写完之后我打了个日志，发现所有被装饰的函数都叫 `wrapper`。

一个两个还看得过去，等有十个函数都套了 `timer`，日志里十行 `wrapper`，我根本不知道慢的是哪个。

修法是加一行：

```python
@functools.wraps(func)
```

它把原函数的名字、说明、类型标注抄到外层这个 `wrapper` 身上。抄完之后 `__name__` 才是对的。

（顺带说，`wraps` 改不了报错里那层壳的名字。Traceback 里照样能看到 `in wrapper`，因为那是代码对象自己的名字，不是属性的名字。这个我一开始也搞混了。）

### 四、`sleep` 放哪

```python
except Exception as e:
    print(...)
    if attempt == times:
        raise
    time.sleep(delay)
```

`sleep` 必须在"是不是最后一次"的判断后面。写在前面的话，最后一次失败之后还得白等半秒，然后才报错。多等的那半秒没有任何用处。

## 测的时候差点漏了

测试一共七条。其中有一条专门测"三次全失败，必须抛错"。

这条最容易被省掉，因为成功路径是主流程，失败路径写起来还得额外造数据。但省掉它，等于没测那句 `raise`。真到了线上，错误被悄悄吞掉，日志干干净净，问题就在那儿躺着。

所以我用了 `pytest.raises`：

```python
with pytest.raises(ValueError):
    always_fail()
```

只要它不抛，这条测试就红。

## 现在

代码在 [github.com/XiaoZege/toolkit](https://github.com/XiaoZege/toolkit)，一共四个东西：计时、重试、日志、临时切目录。每次 push 会自动跑测试。

写之前我以为装饰器是个花招。写完才知道，它解决的是一类很实在的问题：**同一段跟业务无关的代码，被重复写在很多地方。**

重试是这样，计时是这样，打日志也是这样。

一旦你发现自己在复制第二遍，那就是该把它包出去的时候了。
