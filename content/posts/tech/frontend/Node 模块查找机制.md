---
title: "Node模块查找机制"
date: 2022-09-18T18:23:35+08:00
draft: false
layout: posts
---



Node里面的模块系统遵循的是CommonJS规范。

## CommonJS回顾
Commonjs规范 的使用非常简单，主要有模块引用，模块定义，模块标识三个部分。
### 模块引用
```javascript
// 采用require方法引入模块API
var fs =  require('fs')
```
### 模块定义
在模块中存在一个module对象，代表模块儿本身，同时上下文环境提供了一个exports对象用于导出当前模块的方法或变量，并且是唯一导出的出口。同时，exports是module的属性。在Node中一个文件就是一个模块。示例代码:
```javascript
// saveToken.js
const fs =  require('fs')

module.exports = function (filename,readStream){
  return new Promise(resolve=>{
    const writeStream = fs.createWriteStream(filename);
    writeStream.on('finish',()=>{
      setTimeout(resolve,100);
    })
    readStream.pipe(writeStream)
  })
}

```
### 模块标识
模块标识其实就是传递给require()方法的参数，必须是合格小驼峰命名的字符串，或者以.,..开头的相对路经或绝对路径，可以忽略后缀名js。

> Tips：exports、module.exports和export、export default 差异
>
> require: node 和 es6 都支持的引入
> export / import : 只有es6 支持的导出引入
> module.exports / exports: 只有 node 支持的导出

## Node模块
在Node中，模块儿可以分为两大类，一类是Node提供的模块成为核心模块；另一类是用户编写的模块，成为文件模块。
在Node中引入模块，大致会经历这么几个过程：

- 路径分析
- 文件定位
- 编译执行
### 核心模块
核心模块在Node源码编译的过程中，编译进了二进制执行文件中。当Node进程启动时，核心模块儿会直接被加载到内存中，所以核心模块引入时，文件定位和编译执行这两个步骤可以忽略掉，并且在路径分析中会优先判断，所以核心模块的加载速度是最快的。
### 文件模块
文件模块则是在运行中动态加载，需要完整的路径分析，文件定位，编译执行过程，加载速度比核心模块会慢一些。

## Node模块查找机制图
![485cee19-21b5-415b-b568-4708a4246f47](https://raw.githubusercontent.com/Leowuqunqun/img/master/image485cee19-21b5-415b-b568-4708a4246f47.png)
