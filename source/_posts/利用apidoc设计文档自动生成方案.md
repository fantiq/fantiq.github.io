---
title: 利用apidoc设计文档自动生成方案
tags: [apidoc,架构,接口文档,]
categories: [架构,api]
date: 2016-09-10 19:11:40
thumbnailImage: logo.png
thumbnailImagePosition: right
---
使用`apidoc` 可以通过分析代码中的注释生成文档，由于`apidoc`生成的文档是一个单文件
多人开发过程中会导致文件的覆盖，可以通过`git` 代码分支合并的时候通过钩子(web hook)生成文档到文档服务器
文档服务器通过`samba` 服务 实现代码服务器将生成的文档写入文档服务器。由于是静态文件可以使用`node` 来跑
这样避免多项目的时候 需要在文档服务器绑定不同的域名来实现单台文档服务器跑多个项目的接口文档的麻烦
<!-- more -->

# 1. 环境安装
#### 1.1 node + apidoc
apidoc 是node的一个工具 需要先安装node才能使用
###### 1.1.1 安装node
下载win64安装文件[下载地址](https://nodejs.org/dist/v4.4.3/node-v4.4.3-x64.msi) (这里忽略加入PATH的操作 如果需要自行添加)
命令行输入 `node -v` 显示版本号 表示你以成功安装 node
###### 1.1.2 安装apidoc
通过 node 的包管理工具 `npm` 安装 `npm install apidoc -g`
执行命令`apidoc --help` 命令识别标识安装成功
**apidoc命令没有提供查询版本的参数(个人没有发现)**

#### 1.2 samba 服务
为了生成文件方便 使用`samba`服务 具体安装方法 [samba 文件共享](/2016/06/07/linux-centos-samba-%E6%96%87%E4%BB%B6%E5%85%B1%E4%BA%AB/)
#### 1.3 文档自动生成
为了避免 各自生成文档 导致文件覆盖的情况，文档生成的操作不开放给开发者
当然开发者可以生成到本地来检查代码文档注释是否ok
我们在使用`git`做版本控制的时候 开发分支是隔离的，一般会有`master` `release` `dev` 三个主分支
开发者开发完毕会将代码合并到`dev`分支跟客户端对接联调，可以在分支合并处 通过钩子生成文档到文档服务器(一般也会在合并分支的时候通过 jenkins 自动构建代码)

# 2. apidoc 使用简介

apidoc 官网地址 [apidoc 官网](http://apidocjs.com/)
注释要使用**标准的块注释** 基本用法举例
```
/**
 * @api {get} /login 登录
 * @apiGroup User
 * @apiName getLogin
 * @apiVersion 1.0.1
 * @apiDescription 登录接口
 *
 * @apiExample 作者
 * Aimee Aimee@example.com
 *
 * @apiParam {String} username 用户名
 * @apiParam {String} password 密码
 * @apiParamExample http
 * /login?username=fantiq&passwoed=123456
 * @apiSuccess {String} code=0 错误码
 * @apiSuccess {String} message='ok' 接口信息
 * @apiSuccess {Object} data 数据
 * @apiSuccessExample json 返回值
 * {
 *      "code" : 0,
 *      "message" : "ok",
 *      "data" : []
 * }
 * @apiSampleRequest http://www.example.com/
 */
```

文档生成命令
```
apidoc -i [代码路径] -o [生成的接口文档静态文件]
```

# 3. apidoc 语法说明

#### 3.1 基本定义
###### 3.1.1 @api
> @api {method} path name

定义基本的接口请求方式 这个参数**必须存在**

|参数|说明|备注|
|--|--|--|
|method|Http请求方法 `GET POST PUT` 等，不区分大小写|如果支持多种请求方式可以写成`{GET POST}` 但这会导致 `apiSampleRequest` 无法使用|
|path|路由||
|name|接口名称 会显示到文档左侧的导航栏||

使用举例
```
# 单个请求方式
/**
 * @api {get} /login 登录
 */
# 多个请求方式
/**
 * @api {get post} /login 登录
 */
```
###### 3.1.2 @apiGroup
> @apiGroup name

定义接口的分组 分组定义会显示到文档左侧的主导航栏。但是这个分组的值不支持中文
可以通过`@apiDefine` 的形式来使用中文

|参数|说明|备注|
|--|--|--|
|name|分组名称||

使用举例
```
/**
 * @apiGroup User
 */
```
通过apiDefine 使用中文
```
/**
 * @apiDefine GroupName 中文
 */
/**
 * @apiGroup GroupName
 */
```

###### 3.1.3 @apiName
> @apiName name

定义接口内部名称。
默认情况下apidoc会将 请求方式以及路由进行拼接成值 一般不建议设置这个值
一个接口在不同版本中`请求方式`和`路由`都不会发生变化的时候尽量用默认值
如果同一个接口中`请求方式`和`路由`在不同版本发生变化 这个定义就会起到作用

|参数|说明|备注|
|--|--|--|
|name|接口名称 默认是 method+path||

使用举例
```
/**
 * @apiName GetLogin
 */
```

###### 3.1.4 apiVersion
> @apiVersion version

定义接口版本

|参数|说明|备注|
|--|--|--|
|version|支持版本格式 major.minor.patch||

使用举例
```
/**
 * @apiVersion 1.0.1
 */
```

###### 3.1.5 apiDescription
> @apiDescription description

|参数|说明|备注|
|--|--|--|
|description|接口描述||

使用举例
```
/**
 * @apiDescription 这是一个登录接口
 * 接口调用成功会分发 token
 */
```

#### 3.2 参数说明

参数说明定义基本分为两个纬度 `参数类型` `参数形式`
下面通过一个表格说明

|#|参数字段说明|参数举例|
|--|--|--|
|基本参数|@apiParam|@apiParamExample|
|请求头|@apiHeader|@apiHeaderExample|
|成功|@apiSuccess|@apiSuccessExample|
|错误|@apiError|@apiErrorExample|

###### 3.2.1 apiParam
> @apiParam [(group)] [{type{size}}] field descript

定义参数字段 其中 `@apiHeader` `@apiSuccess` `@apiError` 用法也是如此

|参数|说明|备注|
|--|--|--|
|type|参数类型 支持的参数类型有`String Number Boolean Object String[]`|定义参数取值范围 `{Number{1-999}}`|
|field|参数字段名称 |设置默认值 `field=defaultValue` , 设置为可选项 `[field]`|
|descript|参数字段描述||

使用举例
```
/**
 * @apiParam {String} name 名称
 */
```
参数类型取值范围定义
```
# 定义2-5个字符
/**
 * @apiParam {String{2..5}} name 名称
 * @apiParam {Number{111-999}} id 编号
 */
# 定义参数的允许值
/**
 * @apiParam {String="a","b","c"} name 名称
 * @apiParam {Number=1,2,3,4} id 编号
 */
```
参数字段
```
# 设置参数默认值
/**
 * @apiParam {String} name="zzz" 名称
 */
# 设置参数为可选值
/**
 * @apiParam {Number} [id=1] 编号
 */
```


###### 3.2.2 apiParamExample
> @apiParamExample [type] title
> content

定义参数例子 其中 `@apiHeaderExample` `@apiSuccessExample` `@apiErrorExample` 用法也是如此

|参数|说明|备注|
|--|--|--|
|type|内容类型 `json`||
|title|参数名称||
|content|参数例子||

使用举例
```
/**
 * @apiParamExample json 返回成功
 * {
 *      "code" : 0,
 *      "message" : "ok",
 *      "data" : []
 * }
 */
```

#### 3.3 定义代码块
定义代码块 **必须** 在独立的注释块中定义

```
# 定义一个常量
/**
 * @apiDefine NAME 张开发
 */
# 使用
/**
 * @apiGroup NAME
 */
```
```
# 定义代码片段
/**
 * @apiDefine TEST
 * @apiSampleRequest http://www.example.com/
 */
# 使用
/**
 * @api {get} /foo/bar
 * @apiVersion 1.0.1
 * @apiUse TEST
 */
```
#### 3.4 其他
###### 3.4.1 权限
###### 3.4.2 忽略注释
代码注释块中有`@apiIgnore` 声明的话 这个代码块里面的文档注释将不会生成到文档
> 建议这个声明写到注释块的头部
###### 3.4.3 接口作者
`apidoc` 的缺点是没有定义接口作者的地方，一个方法就是利用`@apiExample` 来定义
```
/**
 * @apiExample 作者
 * 张开发 dev@company.com
 */
```

# 4. 文档服务器

建议单独一个服务器跑 文档 ，文档可以通过ip+端口



























<!-- 




开发
    目的：制作产品
基础架构
    目的：开发者之间更好的配合 为开发提供便利
    移动端
        Android
        Ios
        Wap
    接口
    运维

    接口之间的开发框架 底层架构
    App之间的底层架构 底层框架
    服务器的架构 项目需要准备的服务
    客户端与后端数据交互标准

运维
    目的：为产品提供需要的环境支持
测试
    目的：发现产品中存在的问题
项目经理
    目的：保证开发产品与设计产品 吻合
    工具：流程图 代码审查
    进度保证 代码质量保证 产品真实度保证
产品经理
    目的：更好的表达请求 产品
    工具：原型图 交互图 设计图 需求文档
    原型图之间的关联 产品流程 必要的文字说明
 -->

