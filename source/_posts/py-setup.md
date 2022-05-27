---
layout: post
title: "python 模块的构建与发布 setup.py"
date: 2020-11-20
excerpt: "`setuptools` 是 distutils 增强版，不包括在标准库中。其扩展了很多功能，能够帮助开发者更好的创建和分发 Python 包。大部分 Python 用户都会使用更先进的 setuptools 模块..."
categories: 
- python
tags:
- setup.py
comments: true
---

### 应用场景
随着爬虫需求越来越多，python系统也越来越多，有很多重复模块每个系统都要copy一遍，比如：apollo配置中心、注册eureka以及一些工具类。

### setuptools 介绍

`setuptools` 是 distutils 增强版，不包括在标准库中。其扩展了很多功能，能够帮助开发者更好的创建和分发 Python 包。大部分 Python 用户都会使用更先进的 [setuptools](https://setuptools.readthedocs.io/en/latest/) 模块。

```
# coding=utf-8
from setuptools import setup,find_packages
import io
import os

def read(*parts):
    filename = os.path.join(os.path.abspath(os.path.dirname(__file__)), *parts)

    with io.open(filename, encoding='utf-8', mode='rt') as fp:
        return fp.read()


setup(
    name='duiba-ext',  # 应用名
    version='0.0.2',  # 版本号
    author="xxx", # 程序的作者
    author_email="xxx",
    description="兑吧扩展",
    long_description=read('README.md'),
    license="MIT",
    url="xxxx",
    packages=find_packages(),
    data_files=['duiba_ext/common/fake_useragent.json'],
    include_package_data = True,
    install_requires=[
        "requests>=2.25.0",
        "redis>=3.5.3",
        "py_eureka_client>=0.7.6",
        "fake-useragent>=0.1.11",
    ],
    classifiers=[
        # 发展时期,常见的如下
        #   3 - Alpha
        #   4 - Beta
        #   5 - Production/Stable
        'Development Status :: 3 - Alpha',
        # 开发的目标用户
        'Intended Audience :: Developers',
        "Topic :: Software Development :: Libraries :: Python Modules",
        "Programming Language :: Python :: 3.6",
        "Programming Language :: Python :: 3.7",
        "Programming Language :: Python :: 3.8",
        "Programming Language :: Python :: 3.9",
    ],
)
```

各个参数的含义

```
--name 库名称，▲需要注意的是不要大写，不然会有坑
--version (-V) 包版本
--author 程序的作者
--author_email 程序的作者的邮箱地址
--maintainer 维护者
--maintainer_email 维护者的邮箱地址
--url 程序的官网地址
--license 程序的授权信息
--description 程序的简单描述
--long_description 程序的详细描述
--platforms 程序适用的软件平台列表
--classifiers 程序的所属分类列表
--keywords 程序的关键字列表
--packages 需要处理的包目录（包含__init__.py的文件夹） 
--py_modules 需要打包的python文件列表
--download_url 程序的下载地址
--cmdclass 
--data_files 打包时需要打包的数据文件，如图片，配置文件等
--scripts 安装时需要执行的脚步列表
--package_dir 告诉setuptools哪些目录下的文件被映射到哪个源码包。一个例子：package_dir = {'': 'lib'}，表示“root package”中的模块都在lib 目录中。
--requires 定义依赖哪些模块 
--provides定义可以为哪些模块提供依赖 
--find_packages() 对于简单工程来说，手动增加packages参数很容易，刚刚我们用到了这个函数，它默认在和setup.py同一目录下搜索各个含有 __init__.py的包。
--install_requires = ["requests"] 需要安装的依赖包
--entry_points 动态发现服务和插件
```

#### setup.py 需要的模块

```
setuptools>=50.3.2
wheel>=0.35.1
twine>=3.2.0
```

#### setup.py 打包命令

```
python3 setup.py sdist bdist_wheel
```

此处列举一些常用命令：

- build:  

构建安装时所需的所有内容

- build_ext:  

构建扩展，如用 C/C++, Cython 等编写的扩展，在调试时通常加 --inplace 参数，表示原地编译，即生成的扩展与源文件在同样的位置。

- sdist:  

构建源码分发包，在 Windows 下为 zip 格式，Linux 下为 tag.gz 格式 。执行 sdist 命令时，默认会被打包的文件：

```
所有 py_modules 或 packages 指定的源码文件
所有 ext_modules 指定的文件
所有 package_data 或 data_files 指定的文件
所有 scripts 指定的脚本文件
README、README.txt、setup.py 和 setup.cfg文件
该命令构建的包主要用于发布，例如上传到 pypi 上。
```

- bdist:  

构建一个二进制的分发包。

- bdist_egg: 

构建一个 egg 分发包，经常用来替代基于 bdist 生成的模式

- bdist_wheel:  

构建一个 wheel 分发包，egg 包是过时的，whl 包是新的标准

- install:  

安装包到系统环境中。

- develop:  

以开发方式安装包，该命名不会真正的安装包，而是在系统环境中创建一个软链接指向包实际所在目录。这边在修改包之后不用再安装就能生效，便于调试。

#### 发布到私服仓库

需要在当前用户根目录下添加 .pypirc 文件,内容如下：

```
[distutils]
index-servers =
    nexus

[nexus]
repository=https://nexus3.dui88.com/repository/hosted/
username=pythoner
password=xxx
```

执行命令：

```
twine upload -r nexus dist/* # 这里是上传所有文件，每次构建的时候旧的版本也会保留在这个目录下面，上传的时候需要确认下。
```

这样其它系统就可以直接安装该模块了。因为我们使用了nexus私服，如果每次 `pip install` 指定源仓库很麻烦，可以全局设置为私服。方法如下:

修改 ~/.pip/pip.conf (没有就自己创建)， 增加或者修改 index-url 至源

```
[global]
timeout = 60
index-url = https://nexus3.dui88.com/repository/pypi-group/simple
```

### 坑点记录

当程序里有.json或者.txt一些非py文件的时候不会自动打包。这时候就要靠`include_package_data` 和 `data_files` 来指定了。data_files一般写成 {'your_package_name': ["files"]} 或者直接路径。还需要一个 `MANIFEST.in` 文件来明确指定哪些文件需要打到包中。

`MANIFEST.in` 文件内容

```
# Include the README
include *.md

# Include the txt file
# include */*.txt

# Include the data files
recursive-include duiba_ext/common/fake_useragent.json
```







