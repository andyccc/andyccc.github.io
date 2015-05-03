---
title: 快速搭建 markdown 个人 github 博客
date: 2015-06-18 10:14:10
categories: 
-其他
tags:
- github
---


### 快速搭建 markdown 个人 github 博客

摘要：本文以 mkdocs 开源文档工具 + markdown 预发编写文档，最终生成简易个人 github 博客。最终效果如：https://andycc.github.io/

### 文章目录

*   *   *   [零成本 - 30 分钟快速搭建 markdown 个人 github 博客](#30markdowngithub_0)
        *   *   [1. 准备工作](#1_5)
            *   [2. 安装 mkdocs](#2mkdocs_11)
            *   *   [2.1 phyton 原生安装 mkdocs](#21_phytonmkdocs_13)
                *   [2.2 brew 方式安装 mkdocs](#22_brew_mkdocs_41)
            *   [3. 准备 github 工程](#3github_68)
            *   *   [3.1 准备源码工程](#31__74)
                *   [3.2 准备部署工程](#32__113)
                *   [3.3 git 自动部署命令](#33_git_162)

#### 1. 准备工作

*   学习 markdown 语法，写好文档必备
*   学习 mkdocs 基本原理和规范，以便合理规划好建站导航
*   安装好 python 或安装好 homebrew

#### 2. 安装 mkdocs

##### 2.1 phyton 原生安装 mkdocs

需要 [Python](https://www.python.org/) 和 Python package manager [pip](http://pip.readthedocs.org/en/latest/installing.html) 来安装 MkDocs . 可以通过以下命令查看是否安装了上述依赖:

```
$ python --version
Python 2.7.2
$ pip --version
pip 1.5.2
```

MkDocs 支持 Python 2.6, 2.7, 3.3 和 3.4.

使用 pip 安装 `mkdocs` :

```
$ pip install mkdocs
```

`mkdocs`已经安装到你的系统. 运行 `mkdocs help` 以检查是否正确安装.

```
$ mkdocs help
mkdocs [help|new|build|serve|gh-deploy] {options}
```

备注：本人 mac 在安装时，由于 pip3 是 3.8 版本，所以在安装时报错无法安装成功，所以采用下面的 brew 方式。不得不说 brew 真的是极大的方便了开发提效。

##### 2.2 brew 方式安装 mkdocs

brew 安装最为简洁

`brew install mkdocs`

等待几分钟，就自动安装好了。

```
andydeMacBook-Pro:github andy$ mkdocs
Usage: mkdocs [OPTIONS] COMMAND [ARGS]...

  MkDocs - Project documentation with Markdown.

Options:
  -V, --version  Show the version and exit.
  -q, --quiet    Silence warnings
  -v, --verbose  Enable verbose output
  -h, --help     Show this message and exit.

Commands:
  build      Build the MkDocs documentation
  gh-deploy  Deploy your documentation to GitHub Pages
  new        Create a new MkDocs project
  serve      Run the builtin development server
```

#### 3. 准备 github 工程

在该步骤，我们需要创建两个 github 工程。一个是编写源文档的库，在该库下进行文档编写。另一个是 mkdocs 编译后的部署仓库，博客的 html 仓库。当我们编写好源文档库后，对源文档库执行 mkdocs build 命令进行编译，将编译后的 site 站点部署文档拷贝到 github 站点仓库。

具体步骤如下。

##### 3.1 准备源码工程

准备好一个编写 markdown 文档的源文档仓库。

参考：https://github.com/andycc/andycc-source

**1）首先在工作空间建立一个 mkdocs 仓库，比如我选择在~/workspace/github/ 下创建目录**

mkdocs new ${your-docs-name}

这里建议仓库名字用个人博客的站点 +"-source"，比如我的 github 站点是：[andycc.github.io](https://github.com/andycc/andycc.github.io) 那我的源文档仓库名字就用：andycc-source

创建好改目录后，会自动生成 mkdocs 博客目录，如下：

andydeMacBook-Pro:github andy$ ls andycc-source/  
docs mkdocs.yml

**2）当我们创建好 mkdocs 源文档库后，就可以在 docs 目录下进行编写 markdown 文档。**

然后执行：mkdocs build

此时会根据 docs 中的 markdown 文档生成对应的站点静态文件 site

```
andydeMacBook-Pro:github andy$ ls andycc-source/
docs		mkdocs.yml         site
```

site 目录下就是 html 静态文件。例如：

```
andydeMacBook-Pro:github andy$ ls andycc-source/site/
404.html	index.html	sitemap.xml.gz	学习ing
css		js		实践
fonts		search		总结
img		sitemap.xml	推荐
```

##### 3.2 准备部署工程

经过 3.1 准备源码工程步骤后，我们得到一个可以作为 github 个人博客部署的站点 html 静态文件。

**1）创建个人博客站点仓库**

此时我们根据自己的 github 账户名称，创建一个【个人 github 账号. github.io】的仓库，此仓库就是 github 为每个账号生成的默认博客站点，比如我的个人博客站点仓库地址是这个：https://github.com/andycc/andycc.github.io 我的博客访问地址：https://andycc.github.io/

**2）部署博客站点**

将 3.1 步骤使用 mkdocs 编译得到的 site 路径下的 html 静态文件拷贝到【个人 github 账号. github.io】仓库下。

cp -rf andycc-source/site/* andycc.github.io

```
lirongdeMacBook-Pro:github lirong$ ls
	andycc.github.io	andycc-source
```

然后提交 andycc.github.io 仓库内容。

**3）访问博客**

比如我的建成后的博客访问地址是：https://andycc.github.io/

不出意外，你的博客访问地址应该是：https:// 你的 github 账号. github.io

自己动手试试吧。

备注：参照本教程搭建好一个博客，还需要你熟悉编写 markdown 文档的语法，以及学会使用 mkdocs 建站的基本语法。mkdocs 建站命令

```
## Commands

* `mkdocs new [dir-name]` - Create a new project.  初始化一个mkdocs博客项目
* `mkdocs serve` - Start the live-reloading docs server. 启动本地预览
* `mkdocs build` - Build the documentation site. 构建源码为html静态资源
* `mkdocs -h` - Print help message and exit. 查看帮助

## Project layout

    mkdocs.yml    # The configuration file.   mkdocs的配置文件，用来编辑菜单等
    docs/
        index.md  # The documentation homepage.   站点首页
        ...       # Other markdown pages, images and other files.  自定义目录，存放图片等其他文件
```

##### 3.3 git 自动部署命令

鉴于每次部署都要做很多重复的步骤，所以我写了个自动部署的脚本。该脚本主要做 x 件事：

1. 编译源文档库

2. 拷贝 html 静态文件到个人博客仓库

3. 自动提交部署个人博客站点

```
#!/usr/bin/env bash
cd  /Users/andy/workspace/github/andycc-source
mkdocs build
cp -rf /Users/andy/workspace/github/andycc-source/site/ /Users/andy/workspace/github/andycc.github.io/
cd /Users/andy/workspace/github/andycc.github.io
git add .
git commit -m "发布提交"
git push
echo "发布github完成"
```

有需要的按照自己的源文档库和个人博客仓库名称进行修改即可。
