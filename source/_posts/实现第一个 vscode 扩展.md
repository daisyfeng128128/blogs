---
title: 实现第一个 vscode 扩展
date: 2016-01-10 18:20
tags:
- vscode
---


> 提前声明：<br>
> 此处使用的 vscode 版本是0.10.6

vscode 是微软最近弄出来的代码编辑器，基于 Electron ，对于前端程序员来说，颇亲切。

个人觉得，到目前这个版本为止， vscode 还不是很成熟，总体体验上离 sublime 还有一定差距。
<!-- more -->

但是我个人很看重 vscode 的这些点：

* 1、虽然使用 Electron ，但是代码各方面处理还是挺快的，特别是打开比较大的 js 文件，基本不会挂掉，性能堪比 sublime ；
* 2、里面全是 js 系列的东西（虽然加了一层 ts ），对于前端来说，很是亲切，如果成熟到一定程度的话，将会有大把的前端程序员参与插件的开发。相比于 sublime 使用 python ， vscode 真是太爽了，深度定制的时候少了语言的门槛。

目前个人感觉的小缺点有：

* 1、无法代码折叠；
* 2、扩展 API 还不完善，有些比较酷的功能依据现有 API 还无法实现。

废话不对说，走一个插件先。

### 插件功能

对 JavaScript 代码进行检查，基于的检查规则是 `fecs` 。

### 安装必要的东西

> npm install -g yo generator-code

### 生成扩展项目

执行下面的代码：

> yo code

然后会出现这样的选择界面：

![](https://github.com/yibuyisheng/blogs/blob/master/imgs/13.png?raw=true)

选择：

> New Extension (JavaScript)

这样就会生成使用 JavaScript 进行插件开发的项目结构。

后续还会设置扩展的名字（此处设为 test ）、扩展的唯一标识、扩展的描述、扩展的发布者名字、是否初始化为 Git 仓库。根据提示做相应设置就好了。设置完之后就会自动运行 `npm install` ，安装好 vscode 模块。

一切结束之后，你会发现在当前目录下生成了一个叫 `test` 的目录，进入这个目录，下面就有了一堆文件。

### 修改 package.json 文件

更改 `activationEvents` 配置项，设为：

```js
[
    "onLanguage:javascript"
]
```

意思就是在打开 JavaScript 文件的时候会激活这个扩展。

删掉 `contributes` 配置项，此处用不上这个配置。

### 修改 extension.js 文件

这个文件是 package.json 里面 `main` 配置指向的文件，扩展激活的时候会调用这个文件提供的 activate 方法。

对于该扩展，其执行流程为：

* 1、在用户打开 js 文件的时候激活扩展，注册好文件保存的回调方法；
* 2、在用户保存文件的时候，执行 fecs 检查；
* 3、将第二步中检查出的错误和警告等信息显示到编辑器中。

#### 开发过程

在开发扩展的时候，要使用到 `vscode` 和 `fecs` 两个模块：

```js
var vscode = require('vscode');
var fecs = require('fecs');
```

然后注册文件保存的回调函数：

```js
var disposable = vscode.workspace.onDidSaveTextDocument(function (event) {
    // do something while saving
});
context.subscriptions.push(diagnosticCollection);
```

> **注意：**<br>
> 此处 `onDidSaveTextDocument` 返回了一个 disposable 对象，这个对象有一个 `dispose` 方法，在扩展销毁的时候，会调用这个方法。因此，这个对象要事先放到 `context.subscriptions` ，`context` 是 `activate` 方法调用的时候传入的上下文对象。

在这个回调函数里面就可以执行 fecs 检查了：

```js
fecs.check(
    {
        type: 'js',
        name: 'FECS JS',
        _: [event.uri.path],
        stream: false
    },
    function (hasNoError, errors) {
        // the result of check
    }
);
```

`hasNoError` 和 `errors` 表明了检查结果。此处可以忽略 `hasNoError` ，直接将 errors 转换成 vscode 能够展示的错误。

vscode 提供了 `DiagnosticCollection` ，用于向界面上展示错误信息。那么如何操作呢？

首先要拿到一个 `DiagnosticCollection` 对象：

```js
var diagnosticCollection = vscode.languages.createDiagnosticCollection('fecs');
```

然后往这个对象里面塞错误信息：

```js
diagnosticCollection.set(someErrorObject);
```

#### 整合所有代码之后的样子

整个 `extension.js` 的代码整合起来如下：

```js
var path = require('path');
var vscode = require('vscode');
var fecs = require('fecs');

fecs.leadName = 'fecs';

exports.activate = function activate(context) {
    var diagnosticCollection = vscode.languages.createDiagnosticCollection('fecs');
    context.subscriptions.push(diagnosticCollection);

    vscode.workspace.onDidSaveTextDocument(function (event) {
        diagnosticCollection.clear();
        fecs.check(
            {
                type: 'js',
                name: 'FECS JS',
                _: [event.uri.path],
                stream: false
            },
            function (hasNoError, errors) {
                diagnosticCollection.set(convertErrors(errors));
            }
        );
    });
};

function convertErrors(fecsErrors) {
    return fecsErrors.map(function (error) {
        return [
            vscode.Uri.file(error.path),
            error.errors.map(function (fileError) {
                // fecs的行号和列号与vscode有差异。。。。
                var line = fileError.line - 1;
                var column = fileError.column - 1;

                return new vscode.Diagnostic(
                    new vscode.Range(line, column, line, column + 1),
                    `[FECS]: ${fileError.message}  (${fileError.rule})`,
                    {
                        1: vscode.DiagnosticSeverity.Warning,
                        2: vscode.DiagnosticSeverity.Error
                    }[fileError.severity]
                );
            })
        ];
    });
}
```

> **注：** [fecs 是什么？](http://fecs.baidu.com/)
