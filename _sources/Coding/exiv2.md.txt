---
date: 2019-12-13
author: JamzumSum
html_meta:
    keywords: C++, exiv
---

# 使用Exiv2修改图像的EXIF头信息

写了一个下载必应美图的程序. 然而还没完, 我一直想改一下![这个东西](_static/exiv2/example.png)...当时注意力就在这些什么"详细信息"啊, "属性"之类的上面, 搜啊搜啊, 也没找到什么办法...

今天早晨拿两个文件做对比试验, 于是发现修改了详细信息的文件(JPEG)会多出来一个"头部", 恍然大明白(捂脸), 一番搜索知道了这个叫`EXIF`头信息, 于是就有了下面这篇文章.

## 编译[Exiv2](https://www.exiv2.org/)

> 我就不跟你们说我从哪找来这么个玩意儿的事了, 反正就是百度上搜两三个读写EXIF的库, 看看哪个好点...

从Exiv2的官网上下载了`2017msvc64`的build包, 加载到自己的项目里就炸了. 原因很奇怪, 说是没有`std::auto_ptr`这种类型. 当年看*Effective C++* 的时候见过这个类型, 肿么就没有了捏? 网上一查知道, `auto_ptr`这种类型在C++11的时候就已经废弃了, C++17时把它移除了. C++语言标准一直都是17, 所以没有这个类型.

我当时一寻思, 我是不太乐意改语言标准的, 因为我比较喜欢尝试新特性...所以就试着自己编译一下.

### git clone

这步你们都懂罢, 拉一份最新的源码下来, 仓库在官网下载页里有, 我就不贴命令了

### 用VS Cmake编译Exiv2

> 我自己是根本不会用什么cmake什么什么的, 但是我知道VS可以帮我干这个. 以前我编译过`tesseract`, 那时候只知道cmake这么个名 ~~(其实现在也一样)~~...

在exiv2的文件夹里`shift+右键`, 点击`用Visual Studio打开`, 如果你没有就用正常办法...打开之后VS会自动建立cmake缓存, 可以看`输出`选项卡看看VS在做什么.

然后呢, 我这是报了个错, 说是找不到`EXPAT`, 我百度了一下都是英文, 索性也不看了, 就用`vcpkg`装了一个...

```{code-block} shell
vcpkg install expat:x64-windows
```

注意我一直是x64平台编译, 所以只安装64位的, 如果你也用x86, 那还要安装x86版本的.

这里给微软打个广告, [vcpkg](https://docs.microsoft.com/zh-cn/cpp/build/vcpkg?view=vs-2017)虽然平时的项目里不知道为啥不好使, 但是在cmake的时候真的是帮我这种小白好大忙, 缺什么包敲个命令就好了, 不用我自己下载编译什么的.

依赖问题解决, cmake缓存正常, 这时候我去src里面找源码, 发现源码里面居然一个`auto_ptr`都没有??? 可能是从git上拉下来的比较新罢, 我也不知道是怎么回事(不敢相信.jpg).

下一步, 告诉编译器我要C++17标准的, 这个找了半天, 我下面写一段代码, __不保证一定是对的__, 我只能说我编译出来了, 而且用起来没问题.

```{code-block} cmake
include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("/std:c++17" _cpp_17_flag_supported)
if (_cpp_17_flag_supported)
    add_compile_options("/std:c++17")
endif()
```

把上面这段代码附加到`Cmakelists.txt`末尾, 全部生成, 一切正常. 安装exiv2, 完毕.

这些做完了之后不要关VS, 现在只是编译了`x64-Debug`版本的, 到配置管理器里添加一个Release版本的再来一次, 省得发布的时候再折腾.

## 使用Exiv2

> 本来网上有几篇写这个的, 但我为什么还要写一遍呢? 因为网上的(至少是我看见的)都是读EXIF信息, 没有写入的, 为此我绕了个弯子, 所以发出来给大伙瞧瞧写入照比读取要多了什么(你能猜到罢?)

```{code-block} cpp
void BingPicker::addCopyRight(const QString& filepath, const QString& copyright) {
    using namespace Exiv2;
    //这改成unique_ptr了, 发现没?
    Image::UniquePtr image = ImageFactory::open(filepath.toStdString());
    assert(image.get());

    image->readMetadata();
    ExifData ed = image->exifData();
    ed["Exif.Image.Copyright"] = copyright.toStdString();
    image->setExifData(ed);         //note this
    image->writeMetadata();
}
```{code-block}

额, 网上没有写入的例子, 我是在官网文档给的例子里找了一阵子把下面两句补上的. 你猜得没错, 写入是要保存的, `setExifData`和`writeMetadata`就是写入的关键, 之前只顾着找网上现成的抄, 运行了发现根本没写进去, 为此一顿好找...

上面`ExifData::operator[]`返回的是一个引用, 如果没有这个键值会直接新建一个(参考map的`operator[]`), 用法很直觉, 不用担心段错误.

## 结尾

我把浏览器关了, 就不找之前看过啥教程了..这把写的比较详细应该...感谢StackOverflow上提问的老哥, 我在回答里翻出来了cmake切换C++语言标准的代码...感谢网上两篇exiv2读EXIF的教程

照例感谢一万篇教程的作者们.

源码-> [GitHub](https://github.com/JamzumSum/BingPicker), 还有Release版本可供下载哦(滑稽)
