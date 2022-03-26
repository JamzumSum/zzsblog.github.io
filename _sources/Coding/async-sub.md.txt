---
date: 2022-03-26
html_meta:
    keywords: Python, async, regexp
---

# re.sub如何异步替换？

> 这是`解析Qzone3TG`系列的第二篇。代码见 [aioqzone-feed/api/emoji.py][sln]

## Motivation

现在有这样一个需求：

```
这是一段[em]e100[/em]富文本[em]e101[/em]，是emotion_cgi_msgdetail_v6的返回[em]102[/em]
```

把上面这段文本里的表情码换成正常人能看的文字或 emoji。大家一眼就能看出来，得用正则表达式。
这个正则很容易写：{regexp}`\[em\]e(\d+)\[/em\]`，然后可以用 {meth}`re.sub` 的 `repl` 参数实现查询调用：

```python
def sync_query(eid: int) -> str | None: ...

re.sub(r"\[em\]e(\d+)\[/em\]", lambda m: sync_query(int(m.group(1))) or m.group(0), text)
```

不过事情往往不会如人们想象的那样发展。比如：我们这个 [查询函数](https://github.com/JamzumSum/QzEmoji/blob/7c85c5d2fcb01631775fc7f2162ad12d98e485b1/src/qzemoji/__init__.py#L59-L61) 压根不是个同步函数。

事实上，由于 `QzEmoji` `async` 分支使用的是 `sqlalchemy`+`aiosqlite`，异步查询是我们的本意。
所以，如何让 {meth}`re.sub` 支持一个异步的替换函数呢？

## Solution

实际上，我们没办法让 {meth}`re.sub` 支持一个异步的 `async`。不过，我们可以自己写一个 {meth}`sub`。

```python
r, tasks = [], []
base = 0
for i, m in enumerate(pattern.finditer(text)):
    fr, to = m.span(0)
    r.append(text[base:fr])  # save un-replaced string as is
    task = asyncio.create_task(repl(m))
    task.add_done_callback(lambda t, idx=i: r.__setitem__(idx, r[idx] + t.result()))
    tasks.append(task)
    base = to
r.append(text[base:])
```

我们用 {meth}`re.finditer` 找出所有表情码的位置。对于每一个表情码 $e_i = text[from_i, to_i]$ 来说，它的 $from_i$ 到上一个表情码的 $to_{i-1}$ 之间这段串 $p_i = text[to_{i-1}, from_i]$ 是不需要修改的，
直接保存就可以，我们记做 $p_i$；每一个表情码对应一个异步查询任务，它的完成回调是给 $p_i$ 串追加查询结果，即 $p_i += query(e_i)$。最后不要忘了，最后一个表情码之后也可能有不需要改变的文字 $p_{n+1} = text[to_n:]$。

记得生成的任务都要存起来，然后我们 {meth}`asyncio.wait(tasks)` 等待所有任务都结束。在这之后，把 $p_0, ... p_{n+1}$
拼起来就可以了：

```python
if tasks: await asyncio.wait(tasks)
return "".join(r)
```

## 后记

关于性能，我并没有做过验证。不过从经验推测，文本中只有一两个表情码的话，我想异步 {meth}`sub` 的速度赶不上 `re.sub`。不过如果有比较多的表情码的话，异步优势还是能显现的。


[sln]: https://github.com/JamzumSum/aioqzone-feed/blob/207b9724d6e89ac17fbcca0406c00bb297c4f6d2/src/aioqzone_feed/api/emoji.py#L31-L59 "aioqzone-feed/api/emoji.py::sub"
