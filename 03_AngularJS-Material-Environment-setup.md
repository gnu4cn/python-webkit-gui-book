# AngularJS-Material环境的建立

## 关于AngularJS-Material

[AngularJS-Material](https://material.angularjs.org/latest/), 既是一个UI部件框架，又是Google公司的[Material 设计](https://material.google.com/)规格的参考实现。该项目提供了一套可重用的、经良好测试的，可用的基于Material 设计的UI部件。

建立AngularJS-Material环境，需要对[npm](https://www.npmjs.com/)、[bower](https://bower.io/)有所了解，尤其是后者，功能十分强大，有着类似于apt-get那样牛X的特性，可以自动完成各种nodejs包、CSS文件等的下载、依赖关系解决。我们将使用Bower来一步完成环境的建立及更新！

## 安装Bower

> 注意：这里的安装都是假定在Ubuntu GNU/Linux环境下。

- 首先安装 npm
`user@host$sudo apt-get install npm`

- 建立一个符号链接

`user@host$sudo ln -s /usr/bin/nodejs /usr/bin/node`

- 使用npm来安装bower

`user@host$sudo npm install -g bower`

> 这里的`-g`表明是全局安装。

到此Bower安装完成。

## 利用Bower安装AngularJS-Material环境

这里要注意的是，应在项目文件夹下，根据项目的需要来安装相应的nodejs包，一般会安装到以下这些包：

```bash
user@host:~/NetBeansProjects/xxx$bower install 'angular-material#master' --save
user@host:~/NetBeansProjects/xxx$bower install 'angular-route#master' --save
user@host:~/NetBeansProjects/xxx$bower install 'angular-sanitize#master' --save
user@host:~/NetBeansProjects/xxx$bower install 'angular-messages#master' --save
```

> 其中`--save`是要将这些nodejs包及版本号，保存在项目目录下的`bower.json`文件中，以便后面执行`bower update`时，能自动对这些包进行更新。

通过以上步骤就完成了AngularJS-Material环境的建立，就可以在Netbeans中直接将相应JavaScript文件及CSS文件拖入到`index.html`中，进行具体的代码编写了。
