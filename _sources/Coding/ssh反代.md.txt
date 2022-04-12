---
date: 2022-04-12
html_meta:
    keywords: reverse proxy, ssh
---

# 不能连外网的机器怎么搭环境？

## Motivation

有很多服务器是被限制只能在内网访问。不论是 conda, pip, docker 还是 apt, 不能访问公网都是一大难题。
这篇不讲如何直接解决这类问题，我们讲讲怎样快速建立一条反代通道。

## 工具

唯一的要求是，本机需要有正在运行的代理。
这里这个“代理”用的是计算机网络中的基本含义，并没有其他乱七八糟的功能（

以下应该把 `${***_port}` 换成实际的端口号。

```{eval-rst}

.. tabs::

    .. tab:: 本地终端

        .. tabs ::

            .. tab:: 有 ssh 配置

                .. code-block:: shell

                    local$ ssh -NR ${remote_port}:localhost:${local_port} remote

            .. tab:: 无 ssh 配置

                .. hint::

                    ssh 配置相当于 ssh 目的服务器的别名，在支持的程序中可代替地址、端口、私钥路径等。

                .. code-block:: shell

                    local$ ssh -NR ${remote_port}:localhost:${local_port} 10.22.33.10


    .. tab:: 远程终端

        .. code-block:: shell

            remote$ export https_proxy=http://localhost:${remote_port}

```

## 后记

conda, curl 这些都支持 {envvar}`https_proxy` 的配置。
