---
date: "2021-08-05"
author: JamzumSum
html_meta:
    keywords: Python
---

# Python 数值中的下划线

直入主题，看一下矛盾所在：

~~~{code-block} python
float(0_0)  # 0.0
float(0_0_0_0_0_0_0)  # 0.0
~~~

像我这种其他语言转来的、没有细致学习过的同学估计很难接受吧。

再看这样一个例子大概能明白过来：

~~~{code-block} python
float(1_0)  # 10.0
float(12_000) # 12000.0
~~~

Python 数值中的下划线起辅助阅读作用，解析的时候和去掉下划线一样。

> This PEP proposes to extend Python’s syntax and number-from-string constructors so that underscores can be used as visual separators for digit grouping purposes in integral, floating-point and complex number literals.

[PEP 515](https://peps.python.org/pep-0515/), py36+.
