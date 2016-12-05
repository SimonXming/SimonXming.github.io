---
layout: post
title: python 打包
category: 语言
tags: python 打包
keywords: packaging
description:
---

### 前言

其实对于 `setup.py` 和 `setup.cfg` 的关注是从 OpenStack 的源码包中开始的，OpenStack 每个组件的发布时都是一个 tar.gz 包，同样，我们直接从 github 上 clone 代码后也会发现两个文件的存在。当阅读 Nova 或 Ceilometer（其他组件可能也会涉及）的代码时，发现 `setup.cfg` 中内容对于代码的理解有很大的影响。那么，到底 `setup.py` 和 `setup.cfg` 是干什么的？

### setup.py

我们从例子开始。假设你要分发一个叫 foo 的模块，文件名 foo.py，那么 setup.py 内容如下：

```python
from distutils.core import setup
setup(name='foo',
      version='1.0',
      py_modules=['foo'],
      )   
```

然后，运行python setup.py sdist为模块创建一个源码包

```python
root@network:/kong/setup# python setup.py sdist
running sdist
running check
warning: check: missing required meta-data: url
warning: check: missing meta-data: either (author and author_email) or (maintainer and maintainer_email) must be supplied
warning: sdist: manifest template 'MANIFEST.in' does not exist (using default file list)
warning: sdist: standard file not found: should have one of README, README.txt
writing manifest file 'MANIFEST'
creating foo-1.0
making hard links in foo-1.0...
hard linking foo.py -> foo-1.0
hard linking setup.py -> foo-1.0
creating dist
Creating tar archive
removing 'foo-1.0' (and everything under it)
```

在当前目录下，会创建 dist 目录，里面有个文件名为 foo-1.0.tar.gz ，这个就是可以分发的包（如果使用命令 `python setup.py bdist_egg` ，那么会在 dist 目录中生成 `foo-1.0-py2.7.egg` 包，setup.py 中第一句引入需要改为 `from setuptools import setup` ）。使用者拿到这个包后，解压，到 foo-1.0 目录下执行： `python setup.py install` ，那么，foo.py 就会被拷贝到 python 类路径下，可以被导入使用（如果安装是 egg 文件，会把 egg 文件拷贝到 `dist-packages` 目录下）。

对于 Windows，可以执行python setup.py bdist_wininst生成一个 exe 文件；若要生成 RPM 包，执行python setup.py bdist_rpm，但系统必须有 rpm 命令的支持。可以运行下面的命令查看所有格式的支持：

```python
root@network:/kong/setup# python setup.py bdist --help-formats
List of available distribution formats:
  --formats=rpm      RPM distribution
  --formats=gztar    gzip'ed tar file
  --formats=bztar    bzip2'ed tar file
  --formats=ztar     compressed tar file
  --formats=tar      tar file
  --formats=wininst  Windows executable installer
  --formats=zip      ZIP file
  --formats=msi      Microsoft Installer
```

执行 sdist 命令时，默认会打包哪些东西呢？

* 所有由py_modules或packages指定的源码文件
* 所有由ext_modules或libraries指定的 C 源码文件
* 由scripts指定的脚本文件
* 类似于 test/test*.py 的文件
* README.txt 或 README，setup.py，setup.cfg
* 所有package_data或data_files指定的文件

还有一种方式是写一个 manifest template，名为MANIFEST.in，定义如何生成 MANIFEST 文件，内容就是需要包含在分发包中的文件。一个 MANIFEST.in 文件如下：

```in
include *.txt
recursive-include examples *.txt *.py
prune examples/sample?/build
```

### setup.cfg

`setup.cfg` 提供一种方式，可以让包的开发者提供命令的默认选项，同时为用户提供修改的机会。对 `setup.cfg` 的解析，是在 `setup.py` 之后，在命令行执行前。

`setup.cfg` 文件的形式类似于

```shell
[command]
option=value
...
```

其中，command是 Distutils 的命令参数，option是参数选项，可以通过python setup.py --help build_ext方式获取。
> 需要注意的是，比如一个选项是–foo-bar，在 setup.cfg 中必须改成 foo_bar 的格式

符合 Distutils2 的 setup.cfg 有些不同。包含一些 sections：
1、global
定义 Distutils2 的全局选项，可能包含 commands，compilers，setup_hook（定义脚本，在 setup.cfg 被读取后执行，可以修改 setup.cfg 的配置，pbr 就用到了这个）
2、metadata
3、files

* packages_root：根目录
* packages
* modules
* scripts
* extra_files

### Setuptools

上面的 setup.py 和 setup.cfg 都是遵循 python 标准库中的 Distutils，而 setuptools 工具针对 Python 官方的 distutils 做了很多针对性的功能增强，比如依赖检查，动态扩展等。很多高级功能我就不详述了，自己也没有用过，等用的时候再作补充。详情可参见这里。

一个典型的遵循 setuptools 的脚本：

```python
from setuptools import setup, find_packages
setup(
    name = "HelloWorld",
    version = "0.1",
    packages = find_packages(),
    scripts = ['say_hello.py'],
    # Project uses reStructuredText, so ensure that the docutils get
    # installed or upgraded on the target machine
    install_requires = ['docutils>=0.3'],
    package_data = {
        # If any package contains *.txt or *.rst files, include them:
        '': ['*.txt', '*.rst'],
        # include any *.msg files found in the 'hello' package, too:
        'hello': ['*.msg'],
    },
    # metadata for upload to PyPI
    author = "Me",
    author_email = "me@example.com",
    description = "This is an Example Package",
    license = "PSF",
    keywords = "hello world example examples",
    url = "http://example.com/HelloWorld/",   # project home page, if any
    # could also include long_description, download_url, classifiers, etc.
)
```

### 管理依赖

我们写依赖声明的时候需要在 setup.py 中写好抽象依赖（install_requires），在 requirements.txt 中写好具体的依赖，但是我们并不想维护两份依赖文件，这样会让我们很难做好同步。 requirements.txt 可以更好地处理这种情况，我们可以在有 setup.py 的目录里写下一个这样的 requirements.txt

```shell
--index https://pypi.python.org/simple/
-e .
```

这样 pip install -r requirements.txt 可以照常工作，它会先安装该文件路径下的包，然后继续开始解析抽象依赖，结合 –index 选项后转换为具体依赖然后再安装他们。

这个办法可以让我们解决一种类似这样的情形：比如你有两个或两个以上的包在一起开发但是是分开发行的，或者说你有一个尚未发布的包并把它分成了几个部分。如果你的顶层的包 依然仅仅按照 “名字” 来依赖的话，我们依然可以使用 requirements.txt 来安装开发版本的依赖包:

```shell
--index https://pypi.python.org/simple/
-e https://github.com/foo/bar.git#egg=bar
-e .
```

这会首先从 https://github.com/foo/bar.git 来安装包 bar ， 然后进行到第二行 -e . ，开始安装 setup 中的抽象依赖，但是包 bar 已经安装过了， 所以 pip 会跳过安装。
