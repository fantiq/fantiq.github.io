---
title: composer使用指南
tags: [composer,php]
categories: [工具,代码管理]
date: 2016-08-10 09:03:37
thumbnailImage: logo.png
thumbnailImagePosition: right
---
composer 是php中经常使用的包管理器 日常使用中也存在很多使用的注意事项
这里总结下使用composer比较实用的知识点
<!-- more -->
## 1. 实现原理

composer 是一个包管理工具 如同`nodejs`中的`npm` 如同`ruby`中的`gem`。composer 的原理其实就是从github上面下载这些软件包 但是这些软件包文件夹的组织结构 分支管理 版本定义 各不相同 ， composer做为各个软件包到达用户的统一入口门面 作用就是规范统一各个软件包的组织形式

composer运行原理如下图
![运行原理](framework.png)

每个github上面的包 都要打包成固定格式上传到 `packagist.org` 才能被composer客户端下载到
客户端通过包的别名 去 `packagist.org` 查到对应版本的软件包在github上的下载地址
查到后将软件包源代码下载到本地 并会 更新本地的 autoload 文件

客户端的实现都是通过一个 `composer.phar` 来实现的 里面的实现代码是`php` 所以需要php来解释执行

## 2. 安装

php 要开启 `openssl` 扩展(composer文件的下载 是 https 协议)

#### windows
通过php函数下载安装文件 `https://getcomposer.org/installer` 并通过php来执行下载的文件将 `composer.phar` 下载到本地
制作执行脚本 `composer.bat` 并将这个文件所在目录添加到系统环境变量`path`中
这样就可以在cmd中直接执行`composer`命令了
```
# 下载 composer.phar
php -r "readfile('https://getcomposer.org/installer');" | php
# 添加到系统
echo @php "%~dp0composer.phar" %*  > composer.bat
```
#### *unix
```
# 下载 composer.phar
curl -sS https://getcomposer.org/installer | php
# 或者
php -r "readfile('https://getcomposer.org/installer');" | php
# 添加到系统
mv composer.phar /usr/local/bin/composer
```
## 3. 使用

软件包的安装是通过 `composer.json` 中定义的 软件包以及版本 进行下载安装的
一个简单的`composer.json` 文件
```
{
    "require": {
        "包名称1": "版本1",
        "包名称2": "版本2",
        .......
    }
}
```

#### 安装软件包
在当前文件夹执行安装命令
```
composer install
```
也许会等待很长时间 由于国内网络问题 需要更换成中国镜像地址才能提高工作效率
更换下载源 [换源方式参考](http://pkg.phpcomposer.com/)
```
# 在 composer.json 文件中更换
"repositories" : {
    "packagist" : {
        "type" : "composer",
        "url"  : "https://packagist.phpcomposer.com"
    }
}
# 命令行更换 这个命令会自动添加上面的内容到 composer.json 文件
composer config -g repo.packagist composer https://packagist.phpcomposer.com
```

安装完成会生成一个 `composer.lock` 文件
因为在安装个软件包的时候我们可以指定一个具体的软件包版本 也可以指定一个版本范围 如
```
# 指定下载 1,2 到 1,3 的最新版本
pkgname:>=1.2,<1.3
# 指定下载 1.2 版本里面最新的版本但不超过 1.2
# 等同 >=1.2,<1.3
pkgname:1.2.*
# 等同 >=1.3,<2.0
pkgname:~1.3
# 等同 >=1.3.1,<1.4
pkgname:~1.3.1
```
composer在安装的时候找到一个符号我们定义规则的版本进行下载 然后会把这个确定下来的版本号
记录到`composer.lock`文件中

当`composer.lock`不存在的时候 
`composer install` `composer update` 都会下载包文件并生成`composer.lock`

当`composer.lock`已经存在的时候
`composer install` 只会从 `composer.lock` 文件中下载固定的软件版本
不会读取 `composer.json` 文件的内容 所以这个情况 当你需要添加新的包时候 修改`composer.json`
文件然后 `composer install` 并不会下载新的包


`composer update` 会从`composer.json` 中读取包定义 并且查询最新的版本并下载
下载完毕会重写`composer.lock` 文件
这样会带来一个问题就是 不同开发者 使用的包版本不一致

当我们需要添加一个新的软件包的时候需要用
```
composer require "包名称:版本"
```
这样会添加新的包规则到 `composer.json` `composer.lock` 文件 而不是重写这两个文件


## 4. 版本

#### 版本号说明
一个php版本号的例子
```
7.0.6
# 或者
7.0.6.160810_Beta2
```
对上面的版本号进行抽象
```
A.B.C[.D_E ]
```
|#|可选值|取值|名称|意义|
|--|--|--|--|--|
|A|int unsigined|>0|主版本号/大版本号|框架调整 流程变更等大的变动|
|B|int unsigined|>0|次版本号|功能的增加 完善|
|C|int unsigined|>0|小版本号|bug的修复等小修改|
|D|ymd|160810|时间|发布时间|
|E|希腊字母|Base|-|尚未实现功能 demo版本|
|-|-|Alpha|-|主功能实现 尚存在bug|
|-|-|Beta|-|存在一些缺陷 如UI方面的调整|
|-|-|RC|-|发布前的一个过度版|
|-|-|Release|-|正式发布版|

我们常用的一个版本定义是`主版本号.次版本号.小版本号`的形式

#### composer中版本号的定义

##### 1. 定义具体的版本
指定下载 `1.2.3` 这个版本
```
pkgName:1.2.3
```
##### 2. 指定具体的版本范围
指定下载 大于等于 `1.2` 小于 `1.3` 的版本范围中最新版本
```
pkgName:>=1.2,<1.3
```
##### 3. 通配符版本
指定下载 `1.2` 版本下 小版本最新的版本
等效 >=1.2,<1.3

```
pkgName:1.2.*
```
##### 4. 下一个重要版本
```
# 等效 >=1.2.3,<1.3.0
pkgName:~1.2.3

# 等效 >=1.2.0,<2.0.0
pkgName:~1.2
```

## 5. 常用命令
```
composer init
composer install
composer update
composer require "包名称:版本"
```

[参考地址](http://docs.phpcomposer.com/)

## 6. 项目中的使用

#### 各个开发人员 各个分支 确保包版本的一致


##### 初始化项目代码
第一次安装包
```
composer install
```
将 `composer.json` `composer.lock` 提交到版本库git

其他人下载代码(clone) 安装包执行
```
composer install
```

##### 更新/添加新的包
其中一个开发执行
```
composer require "包名称:版本"
composer update "包名称:版本"
```
这个时候 `composer.json` `composer.lock` 都会写入这个新添加的包信息
安装后 将`composer.json` `composer.lock` 这两个文件修改提交到版本库

其他开发将这个两个文件的更新 合并到自己分支代码 执行
```
composer install
```
这个时候是从`composer.lock`安装包 这样所有开发这安装的包版本是一致的
[参考链接](https://phphub.org/topics/1901)

#### 用composer搭建内部包管理系统

###### a 创建私有库/包
使用github做代码仓库(vcs) 下载自建的git代码
```
git clone git@github.com:fantiq/vertu.git
```
将项目初始化成一个 composer 包
```
cd vertu
composer init
# 填写基本信息 生成 `composer.json`
```
![composer init](composer-init.png)
生成的 `composer.json` 文件内容
```
{
    "name": "fantasy/vertu",
    "description": "composer test",
    "type": "composer",
    "license": "GPL",
    "authors": [
        {
            "name": "fantiq",
            "email": "fantiq@163.com"
        }
    ],
    "require": {}
}
```
[库的配置 参考链接](http://docs.phpcomposer.com/02-libraries.html)
创建开发分支 创建发行版本 提交版本库
```
git add .
git commit -m '增加composer配置文件'
git push origin master
# 以版本号(无小版本号) 创建分支 composer会自动将这些分支识别成开发版本
git branch 1.0
git push origin 1.0
git branch 1.1
git push origin 1.1
git branch 1.2
git push origin 1.2
# 以版本号创建 标签 github会对应生成release 并且会自动打包代码
# composer下载的正式版就是这些tag打包的文件
git tag v1.0.3
git tag v1.1.5
git tag v1.2.1
git push origin --tags
```

###### b 搭建本地资源库
本地资源库主要的是 `packages.json` 文件的生成 可以借组 `satis` 来实现
```
# 下载 `satis`
composer create-project composer/satis --stability=dev 
# 生成satis配置文件
php /satis/bin/satis init satis.json
# 提示输入 资源库名称
Repository name: test pkg
# 提示输入 资源库地址
Home page: http://packagist.dev
```
上面命令会生成配置文件 `satis.json`
文件内容大致如下
```
{
    "name" : "自定义资源包名称",
    "homepage" : "资源包域名",
    # 定义资源查询地址 读取对应地址的 composer.json 文件 解析出包的内容
    "repositories" : [
        {
            "type" : "cvs",
            "url" : "git@github.com:fantiq/vertu.git"
        }
    ],
    "require" : {
        # "包名" : "需要下载的版本"
        "fantiq/vertu" : "*"
    },
    # 下载所有
    "require-all" : true
}
```
生成资源库文件
```
php satis/bin/satis build satis.json public/
```
通过http服务器 将配置的资源库域名指向上面文件生成的目录 `public`

[satis 使用的参考链接](http://docs.phpcomposer.com/articles/handling-private-packages-with-satis.html)

###### c 从自建资源库 下载 私有包

```
composer.json
{
    "require" : {
        "fantiq/phpframework" : "*",
        "fantiq/vertu" : "*",
    },
    "repositories": [
        {
            "type" : "composer",
            "url"  : "http://packagist.dev/"
        },
        {
            "type": "composer",
            "url": "https://packagist.phpcomposer.com"
        }
    ]
}

composer install
```

---


原理 ：
当客户端使用composer下载的时候会先从`composer.json` 文件中寻找下载源列表
```
repositories : [
    {"type":"composer", "url":"http://example.org/1"}
    {"type":"composer", "url":"http://example.org/2"}
    ...
]
```
composer会按下载源的顺序寻找要下载的软件包 都找不到 才会通过全局下载源地址寻找包
下载源地址根目录都会有一个`packages.json` 文件 软件包下载地址就是从这个文件中查找的

服务端 资源库 
官方资源库
中国镜像资源库
自建私有资源库

packages.json

```
{
    "packages" : {
        "vendor/package-name" : {
            "dev-master" : {@composer.json},
            "1.0.x-dev" : {@composer.json},
            "1.0.2" : {@composer.json}
        }
    }
}

packages.json 拆分
{
    "includes" : {
        # 单个packages.json 文件的文件名
        "packages1.json" : {
            "sha1" : "525a85fb37edd1ad71040d429928c2c0edec9d17"
        },
        "packages2.json" : {
            "sha1" : "897cde726f8a3918faf27c803b336da223d400dd"
        },
    }
}
```
[资源库相关参考信息](http://docs.phpcomposer.com/05-repositories.html)

客户端 包/库

包/库 <- 资源库 <- 库

composer.json

库 : 一个可以提供给其他`包`通过composer 下载来使用的 `包` , 库可以依赖其他的库
包 : 需要使用其他的库 但是不对其他的`包`提供服务, 包是一个私有项目

库的`composer.json` 文件中需要使用 `name`  声明库名称
包的`composer.json` 文件中仅仅使用 `require` 声明依赖

`composer.json` 文件结构说明 
**说明** json格式中是不支持注释的书写的 这里写的注释只是为了方便说明

```
{
    # 库名称 如果是库 这个字段是必填项
    "name" : "foo/bar",
    # 如果是库 这个字段是必填项
    "description" : "库的功能简介",
    # 库的关键词 在资源库中搜索时候会用到
    "keywords" : ['foo', 'bar'],
    # 库/包 的安装类型 定义安装逻辑 默认值 library
    "type" : "library|project|metapackage|composer-plugin",
    # 库主页地址
    "homepage" : "http://example.org",
    # 库的版本 这个建议填写 composer 会从你的git中的分支 标签分析出版本
    "version" : "1.0.2",
    # 版本发布时间
    "time" : "YYYY-MM-DD HH:MM:SS",
    # 许可协议
    "license" : "MIT|GPL|BSD|Apache",
    # 项目相关人员信息
    "authors" : [
        {
            "name" : "范易龙",
            "email" : "fanyilong@kuaiqiangche.com",
            "homepage" : "https://fantiq.github.io",
            "role" : "Developer"
        },
        {
            "name" : "fantiq",
            "email" : "fantiq@163.com",
            "homepage" : "https://fantiq.github.io",
            "role" : "Translator"
        },
    ],
    # 项目联系方式
    "support" : {
        "email" : "support@example.org",
        "issue" : "https://github.com/fantiq/phpframework/issues",
        "wiki" : "https://github.com/fantiq/phpframework/wiki",
        "forum" : "",
        "source" : "",
        "irc" : "irc://irc.freenode.org/composer"
    },
    # 定义依赖包
    "require" : {
        # 定义版本范围 >=1.2.1,<1.3.0
        # 定义版本发行 1.0.*@beta
        # 精确到一个开发版本的具体commitID dev-master#2eb0c0978d290a1c45346a1955188929cb4e5db7
        "包名称" : "版本号或者版本范围",
        .....
    },
    # 提供开发测试需要的额外包 默认会进行安装
    # install update的时候可以通过参数 --no-dev 来排除这里定义的包
    "require-dev" : {
        "包名称" : "版本号或者版本范围",
        .....
    },
    "conflict" : {},
    "replace" : {},
    "provide" : {},
    # 推荐给使用者的包 建议安装的信息
    "suggest" : {
        "包名称" : "自定义提醒信息"
    },
    "autoload" : {
        "psr-4" : {
            # 在自动加载的时候会将命名空间前缀 替换成这里定义的路径
            "pre-namespace" : "path",
            ...
        },
        "classmap" : [],
        # 需要自动加载进来的文件
        "files" : "foo/bar/file.php"
    },
    "autoload-dev" : {},
    # 添加到 php include_path 列表的路径
    "include-path" : ['foo', 'bar', '..'],
    "target-dir" : {},
    # 稳定性过滤行为 
    "minimum-stability" : "stable|RC|beta|alpha|dev",
    # 指明 优先下载稳定版
    "prefer-stable" : true,
    # 资源库地址定义 composer在查找资源包的时候是根据顺序进行查找的
    "repositories" : [
        {
            "type" : "composer|vcs|pear",
            "url" : "http://pkg.example.org/1"
        },
        {
            "type" : "composer",
            "url" : "http://pkg.example.org/2",
            "options" : {
                "ssl" : {
                    "verify_peer" : "true"
                }
            }
        }
    ],
    "config" : {},
    # 挂载的脚本 可以在composer执行的一些事件点 挂载执行的脚本
    "script" : {
        "post-install-cmd" : [
            "foo\\bar\\baz::func",
            "phpunit -c app/"
        ]
    },
    # 提供给script的数据 脚本中使用方式 $extra = $event->getComposer()->getPackage()->getExtra();
    "extra" : {},
    "bin" : {},
    "archive" : {
        "exclude" : ['/foo/bar', 'baz', '/*.test', '!/foo/bar/baz']
    }

}
```
[composer.json 可用字段参考](http://docs.phpcomposer.com/04-schema.html)