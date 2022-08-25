## Python项目管理

### PyPI

依赖项是指运行Python 项目中所依赖的外部 Python 包。在 Python 中，这些依赖通常可以在 Python 包索引（PyPI）或其他的管理工具中找到（例如 `Nexus`）。

Find, install and publish Python packages with the Python Package Index

#### whl文件

whl格式本质上是一个压缩包，里面包含了py文件，以及经过编译的pyd文件。使得可以在不具备编译环境的情况下，选择合适自己的python环境进行安装。

安装：
```bash
$ pip install some-package.whl

# 本地安装
$ pip install /some-dir/some-file.whl
```



#### 内网python库

方案有下列四种，按简单到复杂排列
 根据实际情况，第一种**，是前三个中最简单的
 **第四种，需要大容量的存贮，暂不考虑

1. pypiserver
2. pip2pi
3. devpi
4. bandersnatch



### requirements.txt：保持环境一致性

Python 项目可能会对某个特定版本的第三方包有依赖。当项目中（至少）有两个依赖项同时依赖于另一个包，而且每一依赖项都需要该外部包的特定版本的情况下，可能会导致依赖冲突的出现。这些情况可以通过包管理工具（例如 `pip`）来处理（但并非都是如此！）。在这种情况下，通常我们需要告诉 `pip` 如何处理依赖关系以及我们需要哪些特定版本。

```bash
# requirements.txt文件示例
$ cat requirements.txt
matplotlib>=2.2
numpy>=1.15.0, <1.21.0
pytest==4.0.1

## 使用requirements.txt初始化环境
$ pip install -r requirements.txt

## 将项目依赖回写到requirements.txt
$ pip freeze
```

一般情况下，我们通过一个 `requirements.txt` 文本文件来指定我们项目的依赖包及其版本。

上面的 `install_requires` 参数可以是一个依赖项列表，以及他们的说明符（包括 `\<`, `>`, `\<=`, `>=`, `==` 和 `!=` 这些运算符）和版本标识符。除此之外，当项目安装时，不在环境中指定的依赖将会从  `PyPI` (默认情况下)下载并安装。

### setuptools：包管理

但如果你想将你代码发布（例如到 `PyPI`上）供他人广泛地使用，仅依靠requirements.txt是不行的。

当你想要发布一个包，你通常需要一些 **元数据**，包括包名称，版本，依赖项，入口点等。而 `setuptools` 提供了执行这些操作的功能。

项目元数据和可选项在 `setup.py` 文件中定义，如下所示：

```python
from setuptools import setup
setup(     
    name='mypackage',
    author='Giorgos Myrianthous',     
    version='0.1',     
    install_requires=[         
        'pandas',         
        'numpy',
        'matplotlib',
    ],
    # ... more options/metadata
)
```

该文件是纯粹的声明性文件，这样写很不优雅。因此，一个更好的方法是在名为 `setup.cfg` 的文件中定义这些可选项和元数据，然后在 `setup.py` 文件中简单地调用 setup() 即可。`setup.cfg` 示例文件如下所示：

```python
[metadata]
name = mypackage
author = Giorgos Myrianthous
version = 0.1[options]
install_requires =
    pandas
    numpy
    matplotlib
```

最终，一个最简单的 `setup.py` 文件，如下所示：

```python
from setuptools import setup
if __name__ == "__main__":
    setup()
```

end

#### wheel：打包

**wheel 是官方现在推荐的打包方式。**

执行成功后，目录下除了 dist 和 *.egg-info 目录外，还有一个 build 目录用于存储打包中间数据。dist 文件夹中 wheel 包的名称如 `firstPKG-0.0.1-py3-none-any.whl`，其中 py3 指明只支持 Python3。

可以使用参数 `--universal`，包名如 `firstPKG-0.0.1-py2.py3-none-any.whl`，表明 wheel 包同时支持 Python2 和 Python3。使用 universal 也成为通用 wheel 包，反之称为纯 wheel 包。

```bash
$ pip install wheel
Collecting wheel
  Downloading wheel-0.37.1-py2.py3-none-any.whl (35 kB)
Installing collected packages: wheel
Successfully installed wheel-0.37.1
WARNING: You are using pip version 21.2.4; however, version 22.2.2 is available.
You should consider upgrading via the 'D:\data_files\py_code\awsomeproject\venv\Scripts\python.exe -m pip install --upgrade pip' command.

## 生成whl文件
$ python setup.py bdist_wheel
running bdist_wheel
running build
installing to build\bdist.win-amd64\wheel
running install
running install_egg_info
running egg_info
writing firstPKG.egg-info\PKG-INFO
writing dependency_links to firstPKG.egg-info\dependency_links.txt
writing top-level names to firstPKG.egg-info\top_level.txt
reading manifest file 'firstPKG.egg-info\SOURCES.txt'
writing manifest file 'firstPKG.egg-info\SOURCES.txt'
Copying firstPKG.egg-info to build\bdist.win-amd64\wheel\.\firstPKG-0.1-py3.10.egg-info
running install_scripts
creating build\bdist.win-amd64\wheel\firstPKG-0.1.dist-info\WHEEL
creating 'dist\firstPKG-0.1-py3-none-any.whl' and adding 'build\bdist.win-amd64\wheel' to it
adding 'firstPKG-0.1.dist-info/METADATA'
adding 'firstPKG-0.1.dist-info/WHEEL'
adding 'firstPKG-0.1.dist-info/top_level.txt'
adding 'firstPKG-0.1.dist-info/RECORD'
removing build\bdist.win-amd64\wheel

```

目录结构

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/python-setuptool.png)

#### twine：上传

wheel 包可以自己使用和传输给其他人使用，但是维护更新不方便，而 PyPI 作为 Python 的软件仓库，可以让所有人方便的上传和下载，以及管理三方库。

首先登录 [PyPI官网](https://pypi.org/)，注册账号。虽然 setuptools 支持使用 `setup.py upload` 上传包文件到 PyPI，但只支持 HTTP 而被新的 twine 取代。

```bash
$ pip install twine$ twine upload dist/*
```

输入 username 和 password 即上传至 PyPI。如果不想每次输入账号密码，可以在当前目录下创建 .pypirc 文件

```python
[distutils]
index-servers =
    local
[local]
repository: http://localhost:2233
username: admin
password: 123456
```

可以上传至内网私有pypi库

#### 使用说明

```bash
$ cat setup.py
from setuptools import setup
if __name__ == "__main__":
    setup()

## 已经没有upload了
$ python setup.py --help
Common commands: (see '--help-commands' for more)

  setup.py build      will build the package underneath 'build/'
  setup.py install    will install the package
...

```



### 应用

首先，让我们了解通常在何时使用两个文件或仅使用两个文件中的一个。**作为一般经验法则**：

- 如果你的包主要用于开发目的，而且你不打算重新发布它，`requirements.txt` 是足够的（即使包是在多台机器上开发的，只用来保障运行环境一致性）。
- 如果软件包仅由你自己开发（即是在单台机器上）但您打算重新发布它，那么 `setup.py`**/**`setup.cfg`就足够了。
- 如果你的包是在多台机器上开发的并且还需要重新发布它，那么将同时需要 `requirements.txt` **和** `setup.py`**/**`setup.cfg` **文件**。

现在，如果你需要同时使用 `requirements.txt` 和 `setup.py`/`setup.cfg` ，那么你需要确保不会重复。

如果你同时使用这两种，你的 `setup.py`（和/或者 `setup.cfg`）文件需要包含抽象依赖项列表，而`requirements.txt` 文件所包含的确切依赖项必须使用`==`来指定限制包版本。

`install_requires`（例如 `setup.py`）定义了单个项目的依赖关系，而 `requirements.txt` 通常用于定义完整 Python 环境的依赖关系。

`install_requires` 要求很少，而 `requirements.txt` 通常需要包含详尽的固定版本列表，以实现完整环境的可重复安装

### 引用

1. https://developer.aliyun.com/article/944352
2. https://www.jianshu.com/p/ea6bdcbc9776
3. https://zhongneng.github.io/2019/01/19/python-setuptools/