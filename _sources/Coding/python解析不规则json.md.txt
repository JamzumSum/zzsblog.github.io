---
date: 2022-03-22
author: JamzumSum
html_meta:
    keywords: js, Python
---

# Python 如何解析不规则 JSON？

> `解析Qzone3TG`系列第一篇，不规则 JSON 解析器。

## Motivation

`aioqzone`有一个包 `jssupport`，专门处理 Qzone 各种 js-style 的返回值。其中有一个模块叫 `jsjson`，用于解析 Qzone 返回的 js字典对象。

```js
{
    html: "<html></html>",
    code: 0,
    hasmore: true,
    merge: [undefined]
}
```

如上，这种格式大家一定会想到 `JSON`, 毕竟 {meth}`json.loads` 实在深入人心。不过上面这段数据并非规范的 JSON，最主要的特征就是键值虽说是字符串，但没有引号。这样的数据直接使用 {meth}`json.loads` 是会报解析错误的。那到底怎么解析这些数据呢？下文给出我曾经使用过的四种方法。

## demjson

如果你在百度上找 [python解析不规则JSON](https://www.baidu.com/s?ie=UTF-8&wd=python%E8%A7%A3%E6%9E%90%E4%B8%8D%E8%A7%84%E5%88%99JSON)，那么很多作者都会告诉你用 `demjson`。我用 `demjson` 用了很长时间，直到感觉爬虫过于缓慢。一番 profiling 之后压力来到了 `demjson` 这边。翻了一下[仓库](https://github.com/dmeranda/demjson)，最后一次 commit 是15年；issue 和 PR 也都有年头了。这促使我去自己想办法解决这个问题。

> PS: `demjson`有一些 fork，比如 [demjson3](https://pypi.org/project/demjson3/)，并没有使用过，不太清楚性能如何。大家可以试试，毕竟 python 不太鼓励造轮子。

评分：
- 代码规模：A
- 速度：D
- 可维护性：D

## 手写 JSON 解析器

JSON 是一种比较简单的语法，这意味着（用python）写一个解析器其实也不太费劲。我在 [Qzone2TG](https://github.com/JamzumSum/QQQR/blob/6e3981e426f4b68f7a9dbd8df23d73e17112e0a6/src/jssupport/jsjson.py) 里使用了这种方案，提速大概几百倍吧，具体多少我忘了。不过这种方案有一定问题，比如：

1. 我不专业。尽管我有一点编译器知识，但我的思路仍然是 naive 的；写出来的解析器很难看，也很不规范。能用应该是唯一的优点。
2. 性能问题。作为一个曾经的C++用户，用 python 写这种代码让我非常难受，感觉性能上存在巨大的提升空间。
3. 维护问题。这份代码只有我自己在用，而我发誓写完之后就再也不看这一坨东西了。那么，谁来维护呢？

评分：
- 代码规模：D
- 速度：A
- 可维护性：D

## Stringify

前文我们已经提到，这段数据其实是个 js字典。那么显然 `Node` 可以直接解析这段数据，然后通过 `JSON` 格式与 Python 交换。由于 `jssupport` 包里有与 node 通信的代码，那么我可以简单的复用一下：

```python
from .execjs import ExecJS

class NodeLoader:
    jsonStringify = ExecJS(js="").bind("JSON.stringify", new=False)

    @classmethod
    async def json_loads(
        cls, js: str, try_load_first: bool = True, parser: Callable[[str], JsonValue] = json.loads
    ) -> JsonValue:
        """
        This function converts a string representation of JS/JSON data into a Python object.
        It may use :node:meth:`JSON.stringify` to convert js to json.

        :param js: Used to Pass in the json string.
        :param try_load_first: Used to Specify whether to try loading the json string with the `parser` first.
        :param parser: Used to Specify the function that will be used to parse the string.
        :return: A python object that represents the same content as the js/json string.
        """
        if try_load_first:
            try:
                return parser(js)
            except json.JSONDecodeError:
                pass

        json_str = await cls.jsonStringify(js, asis=True)
        try:
            return parser(json_str)
        except json.JSONDecodeError as e:
            logger.exception("Failed to decode json input!")
            logger.debug("json_str=%s", json_str)
            raise e
```

这段代码很好懂，就是借助 `JSON.stringify` 这个 node 函数把 js 转化为 JSON 然后再读取。不过问题来了：
1. 进程间通信。启动一个node进程有比较大的资源开销，进程间通信也不是一个特别高效的手段。
2. 平台限制。在不同的平台上，`subprocess.PIPE` 有不同的缓冲区大小限制。尽管我在 windows 上没遇到任何问题，但我的数据在 Docker 容器（Linux）内被截断了。在这方面，查找资料很困难。
3. 我们从算法的角度考虑：和正常的 JSON 解析器相比，这种方法多读取了一遍字符串。这似乎不是很合理。

评分：
- 代码规模：A-
- 速度：B
- 可维护性：B

> 问题2是个很复杂的点，如果我有朝一日弄懂了可以考虑再水一篇（

## ast

这是目前 `aioqzone` 的默认方法，不过 [Stringify](#stringify) 也被保留了下来。让我们观察数据：如果不考虑变量是否定义、关键字不同等等问题，一个**js 字典似乎和 python 字典没什么两样**。Python 内置 `ast` 库用于解析 python 代码。是不是可以利用它解析 js 字典呢？

```python
class RewriteUndef(ast.NodeTransformer):
    def __init__(self) -> None:
        if int(python_version_tuple()[1]) < 8:
            # NOTE: visit_Constant not available on py37
            self.visit_Str = lambda node: ast.Str(s=node.s.replace("\\/", "/"))

    const = {
        "undefined": ast.Constant(value=None),
        "null": ast.Constant(value=None),
        "true": ast.Constant(value=True),
        "false": ast.Constant(value=False),
    }

    def visit_Name(self, node: ast.Name):
        if node.id in self.const:
            return self.const[node.id]
        return ast.Str(s=node.id)

    def visit_Constant(self, node: ast.Constant) -> Union[ast.Constant, ast.Str]:
        if not isinstance(node.value, str):
            return node
        return ast.Str(s=node.value.replace("\\/", "/"))
```

当我们解析一段“Python代码”得到语法树之后，{class}`ast.NodeTransformer`可以用于修改这棵树。我们知道，输入的 js 中其实没有任何变量，因此任何被识别为变量的 token 其实只是没有引号的字符串而已。因此我们在 `visit_Name` 中把变量全换成字符串；关键字方面，如果有名叫 `true`, `false`, `undefined`, `null` 的变量，那么应该把他们换成对应的常量。最后，针对 JSON 和 Python 转义规则不一样的问题，需要一个替换把不必要的转义符去掉。

> py37 支持：`visit_Constant`是 py38 加入的接口，为了支持 py37 需要使用 `visit_Str`.

```python
from textwrap import dedent

class AstLoader:
    class RewriteUndef(ast.NodeTransformer): ...

    @classmethod
    def json_loads(cls, js: str, filename: str = "stdin") -> JsonValue:
        """
        The json_loads function loads a JSON object from a js/json string. It uses standard
        :mod:`ast` module to parse the js/json.

        :param js: Used to Pass the js/json string to be parsed.
        :param filename: Used to Specify the name of the file that is being read. This is only for debug use.
        :return: A jsonvalue object.
        """

        node = ast.parse(dedent(js), mode="eval")
        node = ast.fix_missing_locations(cls.RewriteUndef().visit(node))
        code = compile(node, filename, mode="eval")
        return eval(code)
```

如上，这段代码肯定**不支持解析所有的不规则 JSON**，但对于解析 Qzone 的返回值来说，它很好用。我使用 `feeds3_html_more` 的返回值来测试它的准确性和性能，从速度上来说，它和 [解析器方法](#手写-json-解析器)相差无几，是[Stringify](#stringify) 的大概十倍。重要的是，我们使用 python 内置库来处理了这个问题，这意味着这个方案能享受到 python
版本优化带来的性能提升。多年以前我似乎见过有人使用 `ast` 来处理不规则 json，但我相信这种方法不是主流。

评分：
- 代码规模：A-
- 速度：A
- 可维护性：A

## 后记

在 [ast法](#ast) 出问题之前，我想没必要再找一个新法子了。在这方面，有一些第三方库也值得关注，比如 [pyjson5](https://github.com/dpranke/pyjson5), [demjson3](https://pypi.org/project/demjson3/)等。不过，等出了事再说换吧（摆烂
