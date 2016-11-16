# 引入AngularJS

上文[怎样利用HTML来建立Python图形用户界面](01_HOWTO_Create_Python_GUIs_using_HTML_zh_CN.md)，可以说具有理论基础的作用，打通了python与webkit之间消息交换的障碍，为我们下一步建立实用的应用程序，奠定了基础。

但为了实际、快速地建立美观实用的应用程序，在有了webkit浏览器引擎后，就可以利用上各种前端框架，比如AngularJS、Angular-Material等，令到应用程序具备响应式布局及非常成熟的用户界面。

这里有必要对上一篇中的代码进行修改，解决一些问题。当然只需对`webgui.py`这个文件进行修改，因为其已具备一个python库模块的功能了。

## 对`webgui.py`的修改

修改后的代码如下，其中有相应说明。

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import time
import Queue
import thread
import urllib

import gtk
import gobject

try:
    import webkit
    have_webkit = True
except:
    have_webkit = False

class UseWebKit: pass

if have_webkit:
    use = UseWebKit
else:
    raise Exception('Failed to load webkit modules')

class WebKitMethods(object):

    @staticmethod
    def create_browser():
        view = webkit.WebView()
        settings = view.get_settings()
        settings.set_property('enable-file-access-from-file-uris', 1)
        return view

    @staticmethod
    def inject_javascript(browser, app, script):
        # 这里对原程式进行了Hack, 简化AngularJS中execute_script的操作
        instruction = "angular.element(document.querySelector('[ng-app=%s]')).scope().%s()"
        browser.execute_script(instruction % (app, script))

    @staticmethod
    def connect_title_changed(browser, callback):
        def callback_wrapper(widget, frame, title): callback(title)
        browser.connect('title-changed', callback_wrapper)

    @staticmethod
    def open_uri(browser, uri):
        browser.open(uri)

implementation = WebKitMethods

def asynchronous_gtk_message(fun):

    def worker((function, args, kwargs)):
        apply(function, args, kwargs)

    def fun2(*args, **kwargs):
        gobject.idle_add(worker, (fun, args, kwargs))

    return fun2

def synchronous_gtk_message(fun):

    class NoResult: pass

    def worker((R, function, args, kwargs)):
        R.result = apply(function, args, kwargs)

    def fun2(*args, **kwargs):
        class R: result = NoResult
        gobject.idle_add(worker, (R, fun, args, kwargs))
        while R.result is NoResult: time.sleep(0.01)
        return R.result

    return fun2

def launch_browser(uri, quit_function=None, echo=False):

    window = gtk.Window()
    window.set_default_size(800, 600)

    box = gtk.VBox(homogeneous=False, spacing=0)

    browser = implementation.create_browser()
    box.pack_start(browser, expand=True, fill=True, padding=0)

    window.add(box)

    if quit_function is not None:
        # Obligatory "File: Quit" menu
        # {
        quit_item = gtk.MenuItem('Quit')
        quit_item.connect('activate', quit_function)
        quit_item.show()

        file_menu = gtk.Menu()
        file_menu.append(quit_item)

        file_item = gtk.MenuItem('File')
        file_item.set_submenu(file_menu)
        file_item.show()

        menu_bar = gtk.MenuBar()
        menu_bar.append(file_item)
        menu_bar.show()

        box.pack_start(menu_bar, expand=False, fill=True, padding=0)

        accel_group = gtk.AccelGroup()
        quit_item.add_accelerator('activate',
                                  accel_group,
                                  ord('Q'),
                                  gtk.gdk.CONTROL_MASK,
                                  gtk.ACCEL_VISIBLE)
        window.add_accel_group(accel_group)
        #
        # }

    if quit_function is not None:
        window.connect('destroy', quit_function)

    window.show_all()

    message_queue = Queue.Queue()

    def title_changed(title):
        if title != 'null': message_queue.put(title)

    implementation.connect_title_changed(browser, title_changed)

    implementation.open_uri(browser, uri)

    def web_recv():
        if message_queue.empty():
            return None
        else:
            msg = message_queue.get()
            if echo: print '>>>', msg
            return msg

    def web_send(app, msg):
        if echo: print '<<<', msg
        asynchronous_gtk_message(implementation.inject_javascript)(browser, app, msg)

    return browser, web_recv, web_send

def start_gtk_thread():
    # Start GTK in its own thread:
    gtk.gdk.threads_init()
    thread.start_new_thread(gtk.main, ())

def kill_gtk_thread():
    asynchronous_gtk_message(gtk.main_quit)()
```

## 对`demo.py`的修改

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
import signal
import os
import time
import urllib

from simplejson import dumps as to_json
from simplejson import loads as from_json

from webgui import start_gtk_thread
from webgui import launch_browser
from webgui import synchronous_gtk_message
from webgui import asynchronous_gtk_message
from webgui import kill_gtk_thread

class Global(object):
    quit = False
    @classmethod
    def set_quit(cls, *args, **kwargs):
        cls.quit = True

def main():
    APP = "webkit_gui_demo"

    start_gtk_thread()

    # Create a proper file:// URL pointing to demo.xhtml:
    file = os.path.abspath('public_html/index.html')
    uri = 'file://' + urllib.pathname2url(file)
    browser, web_recv, web_send = \
        synchronous_gtk_message(launch_browser)(uri,
                                                quit_function=Global.set_quit)

    # Finally, here is our personalized main loop, 100% friendly
    # with "select" (although I am not using select here)!:
    while not Global.quit:

        current_time = time.time()
        again = False
        msg = web_recv()

        if msg:
            msg = from_json(msg)
            again = True

        if msg == "button-clicked":
            # 这里第一个参数是AngularJS的`ng-app`属性值，第二个是定义在AngularJS控件中的
            # 函数名称。
            web_send(APP, "update_desc")

        if again: pass
        else:     time.sleep(0.1)


def my_quit_wrapper(fun):

    signal.signal(signal.SIGQUIT, Global.set_quit)
    signal.signal(signal.SIGINT, Global.set_quit)

    def fun2(*args, **kwargs):
        try:
            x = fun(*args, **kwargs) # equivalent to "apply"
        finally:
            kill_gtk_thread()
            Global.set_quit()
        return x
    return fun2

if __name__ == '__main__': # <-- this line is optional
    my_quit_wrapper(main)()
```

## AngularJS中对`web_recv`及`web_send`的适配

这里将第一篇文章中的`ipc.js`，封装成了业务逻辑服务`sendMsg`，于是可方便的在应用的多处加以注入与调用。

`bservices.js`:

```javascript
'use strict';

var bServices = angular.module('bussinessServices', []);

bServices.factory('sendMsg', ['$document', 
    function($document){
        return function(msg){
            $document[0].title = "null";
            $document[0].title = msg;
        };
    }]);
```

而接收`web_send`消息的函数，编写在控件中，对页面的控制十分完全，同时逻辑也很简单。

`controllers.js`:

```javascript
'use strict';

var Controllers = angular.module('Controllers', []);

Controllers.controller('MainCtrl', ['$scope', 'sendMsg',
    function MainCtrl($scope, sendMsg) {
        $scope.btn_text = "按 钮";
        $scope.card_head = "测试图片";
        $scope.image_desc = "这是一张测试图片";
        var flag = 0;
        $scope.btnClicked = function () {
            sendMsg('"button-clicked"');
        };
        $scope.btnHeadClicked = function () {
            sendMsg('"button-head-clicked"');
        };
        $scope.update_desc = function () {
            // 这里使用 $scope.$apply, 为的是令到web_send过来的消息立即
            // 得到处理，避免延迟。至于为什么要这样做，请自行google 
            // 'angularjs scope apply'
            $scope.$apply(
                    function () {
                        if (flag === 0) {
                            $scope.image_desc = "This is a test pic." + " " + flag.toString();
                            flag = 1;
                        } else {
                            $scope.image_desc = "这是一张测试图片" + " " + flag.toString();
                            flag = 0;
                        }
                    });
        };
        $scope.update_head = function () {
            $scope.$apply(
                    function () {
                        if (flag === 0) {
                            $scope.card_head = "TEST PIC" + " " + flag.toString();
                            flag = 1;
                        } else {
                            $scope.card_head = "测试图片" + " " + flag.toString();
                            flag = 0;
                        }
                    });
        }
    }]);
```

## 问题
暂无问题。
