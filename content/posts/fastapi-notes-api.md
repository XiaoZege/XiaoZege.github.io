---
title: "用 FastAPI 写了五个接口，然后眼睁睁看着数据全没了"
date: 2026-11-13
draft: false
tags: ["Python", "FastAPI", "后端"]
categories: ["项目记录"]
ShowToc: true
---

这周要做一个 API。动手之前我连"接口"是啥都说不清楚，只知道简历上都写"熟悉接口开发"。

做完之后明白了：接口就是别人能通过一个网址来操作你程序里数据的东西。我这周做的，是让别人能通过网址给一个笔记本加一条笔记、删一条、改一条、看全部。

## 五个动作就够了

做出来的东西长这样：

| 网址 | 干什么 |
| --- | --- |
| `GET /notes` | 看全部 |
| `GET /notes/3` | 看第 3 条 |
| `POST /notes` | 加一条 |
| `PUT /notes/3` | 改第 3 条 |
| `DELETE /notes/3` | 删第 3 条 |

统一的规律是：**网址指"哪个东西"，前面的动词指"对它干什么"**。

`GET` 是看，`POST` 是放，`PUT` 是换，`DELETE` 是拿掉。同一个 `/notes/3`，配上不同动词就是四种操作。

代码里它们长得几乎一样：

```python
@app.get("/notes")
def list_notes(limit: int = 10, offset: int = 0):
    return notes[offset : offset + limit]

@app.delete("/notes/{note_id}", status_code=204)
def delete_note(note_id: int):
    for i, note in enumerate(notes):
        if note.id == note_id:
            notes.pop(i)
            return
    raise HTTPException(status_code=404, detail="笔记不存在")
```

`@app.get` 和 `@app.delete` 是装饰器。上周学的那玩意儿，这周第一次派上正经用场——它的作用是把函数登记到一张"网址对应哪个函数"的表里。

## 最惊艳的是一行前端都没写

跑起来之后，浏览器打开 `http://127.0.0.1:8000/docs`,出来一个能点开、能填参数、能直接执行请求的页面。

上面列着我写的每一个接口，参数框、请求体格式、可能返回的状态码，全都标好了。

我一行 HTML 没写，一份接口文档就有了。

后来才想明白它是怎么来的：**它读的是我写的类型标注。**

```python
def get_note(note_id: int):
```

标了 `int`，它就知道这是网址里的一段数字，于是在文档页面上给我一个整数输入框，还在背后自动做了校验。我请求 `/notes/abc`，它直接回 422，函数根本不会被调用。

上周学 `typing` 的时候，老师说标注在运行时不起作用，只是一份说明书。放到 FastAPI 手里，说明书变成了功能。

## 没写一行校验，它自己把人拦住了

`POST /notes` 收的数据长这样：

```python
class NoteIn(BaseModel):
    title: str
    content: str = ""
```

`title` 后面没有 `=`，意思是必须有。`content` 有 `= ""`，意思是可不给。

我故意试了一次，只交 `content`、不交 `title`：

```json
{"content": "我没有标题"}
```

结果 422，返回里写得清清楚楚：

```
"loc": ["body", "title"],
"msg": "Field required"
```

我一行判断都没写。是 `BaseModel` 在拿到数据那一刻自己对着清单查的，缺了就退回。

这跟我之前学的 dataclass 不是一回事。dataclass 只在类型检查工具里报错，运行时不管；`BaseModel` 是真的在运行的时候拦人。

我还试了另一个，`title` 传数字：

```json
{"title": 123}
```

我以为它会把 123 转成 `"123"`，因为网上很多教程这么讲。**结果被拒了**，说 `Input should be a valid string`。

查了才知道，那是 Pydantic v1 的行为，v2 把这个宽松处理去掉了。照着老教程写，很容易对不上。

## 然后数据没了

做到分页那步，我改了 `list_notes` 的签名，加两个参数：

```python
def list_notes(limit: int = 10, offset: int = 0):
```

保存，回浏览器试：

```
/notes?limit=2
```

返回 `[]`。

我以为分页写错了。把 `?limit=2` 去掉，直接 `/notes`，还是 `[]`。

四条笔记，一条不剩。

折腾了一会儿才反应过来：**是我保存文件导致的**。

启动命令里带了 `--reload`，它的作用是：文件一改，自动重启程序。而程序一重启，这行就会被重新执行：

```python
notes: list[Note] = []
```

每次启动都造一个全新的空列表。之前那四条，存在内存里，跟着旧进程一起没了。

不是我分页写错了，是我**用内存存数据**这件事本身有天花板。别说改代码，就是按 `Ctrl+C` 停掉再重新跑，数据一样清空。

## 所以下一步

这周做的东西能跑、能用、文档自动生成，但它记不住任何东西。

下周要换 MySQL，把这些数据写到硬盘上。重启、关机、断电，数据都得在。

本来我以为数据库是个"高级话题"，现在知道它解决的是一个特别朴素的问题：**内存留不住东西。**

写完这五个接口，那些"建表""索引""事务"终于有了着落——它们不是为了考试，是为了让这些数据活下来。

代码在 [github.com/XiaoZege/notes-api](https://github.com/XiaoZege/notes-api)。
