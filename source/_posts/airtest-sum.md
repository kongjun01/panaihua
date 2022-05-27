---
layout: post
title: "使用Airtest爬虫总结和注意事项"
date: 2020-11-12
excerpt: "一开始调研的时候使用的就是Appium，功能全，文档也很多，而且元素定位方式比Airtest靠谱很多。我们要爬的是美团的活动广告页，开始使用appinum都很顺利，但是遇到webview的时候，它的页面是canvas的结构，根本没办法使用元素定位，所以这个时候Airtest的图像定位就起作用了。"
categories: 
- 爬虫
tags:
- Airtest
comments: true
---

##### 为什么不使用Appium
一开始调研的时候使用的就是Appium，功能全，文档也很多，而且元素定位方式比Airtest靠谱很多。我们要爬的是美团的活动广告页，开始使用appinum都很顺利，但是遇到webview的时候，它的页面是canvas的结构，根本没办法使用元素定位，所以这个时候Airtest的图像定位就起作用了。

##### webview如何开启debug模式
网上的大部分教程都是手动修改app的源码（[官网文档](https://developers.google.com/web/tools/chrome-devtools/remote-debugging/webviews?hl=zh-cn)），但是第三方总不会给你个debug的包吧。

Android有个一个很牛逼的软件[太极](https://taichi.cool/zh/download.html)，手机不需要Root就可以运行Xposed框架。里面有很多模块都很不错，比如钉钉助手、抢红包，有兴趣的同学可以试试，一定会屡试不爽的。  
所以，webview开启debug也有模块，[查看地址](https://github.com/taichi-framework/TaiChi/issues/805)

##### 巧用keyevent("BACK")替代返回的截图脚本
很多时候，我们需要从APP的某个页面，回到APP首页，一些同学可能会使用一堆的返回图标的截图语句，来实现这个需求：  

![](/images/125b98339a0425e.jpeg)  
实际上，如果同学们测的是安卓设备，完全可以用 keyevent("BACK") 来替代这个返回的截图语句，更加稳定高效：  

![](/images/965e175d5b414762.image)

##### 局部截图保存到指定位置
局部截图或者说按坐标截图是大家经常会问到的问题，Airtest提供了 crop_image(img, rect) 方法可以帮助我们实现局部截图，但是[官方demo](https://airtest.doc.io.netease.com/IDEdocs/faq/4_common%20problems/#8)并没有给出保存到指定位置：

举个例子，我们想要截取手机屏幕中被红框圈中位置的截图：  
![](/images/aHR0cHM6Ly9ub3Rl.png)

```python
# -*- encoding=utf8 -*-
__author__ = "AirtestProject"

from airtest.core.api import *
# crop_image()方法在airtest.aircv中，需要引入
from airtest.aircv import *

auto_setup(__file__)

# 局部截图
screen = G.DEVICE.snapshot() #这一步之后不能打开其它页面，因为是截取当前页面
screen = aircv.crop_image(screen,(0,160,1067,551))
# 保存局部截图到log文件夹中
filename = "%(time)d.jpg" % {'time': time.time() * 1000}
filepath = os.path.join('需要保存的路径', filename)
aircv.imwrite(filepath, screen, quality, max_size=max_size)
```

##### 如何在flaskweb项目中运行
首先要知道如何脱离IDEA在python环境下直接运行 [官网教程](https://airtest.doc.io.netease.com/IDEdocs/run_script/1_useCommand_runScript/#python)

比如使用apscheduler定时运行airtest脚本

```python
import os

devices = [{'name': '34235ed0', 'location': '北京'}, {'name': '2a5b398e0005', 'location': '上海'}]
comand = "airtest run {} --device android://127.0.0.1:5037/{} --log ~/logs/berserker/meituan"
air_path = '{}/airtest.shell/meituan.air'.format(os.getcwd())

def run():
    for device in devices:
        os.system(comand.format(air_path, device.get('name')))
```


##### 搭建手机爬虫集群
一台电脑可以连接三十台手机，那么如果有很多电脑和很多手机，就可以实现手机爬虫集群，其运行效果如下图所示。
!()[/images/ccdf080c7af7e8a1.png]

可以使用手机的无线调试模式，首先想到的是搭建手机管理平台 [STF](https://github.com/openstf/stf)
获取adb的url代码如下:

```python
# 获取手机的adb地址
async def take_up_device(device=''):
    async with aiohttp.ClientSession(connector=TCPConnector(verify_ssl=False), timeout=timeout) as session:
        headers = {'Authorization': 'Bearer {}'.format(cgf.get('yunce', 'auth_token'))}
        result = await async_utils.form_post(session, take_url, is_re_json=True, headers=headers,
                                             json_data={'serial': device, 'timeout': 600000})
        if not result.get('success', False):
            app_logger.error('获取设备异常，错误信息：%s', result.get('description'))
            return

        result = await async_utils.fetch(session, adb_url.format(serial=device), is_re_json=True,
                                             headers=headers)

        if not result.get('success', False):
            app_logger.error('获取设备adb链接错误，错误信息：%s', result.get('description'))
            return

        return result.get('device', {}).get('remoteConnectUrl')
```

有了这些，是不是就可以搞个刷抖音平台？

##### 结尾

附上爬取美团活动的代码：

```python
# -*- encoding=utf8 -*-
__author__ = "panaihua"

from airtest.core.api import *
from poco.drivers.android.uiautomation import AndroidUiautomationPoco
import logging
import os
import time

logger = logging.getLogger("airtest")
logger.setLevel(logging.ERROR)

auto_setup(__file__)

# connect_device("android://127.0.0.1:5037/172.16.140.68:17593?cap_method=MINICAP_STREAM&&ori_method=MINICAPORI&&touch_method=MINITOUCH")

poco = AndroidUiautomationPoco(use_airtest_input=True, screenshot_each_action=False)

wake()

stop_app('com.sankuai.meituan')

home()

start_app('com.sankuai.meituan')

sleep(10)
poco("免费领水果").click()

sleep(10)

if exists(Template(r"tpl1604041772025.png", record_pos=(0.017, 0.381), resolution=(1080, 2160))):
    touch(Template(r"tpl1604041788888.png", record_pos=(0.381, -0.484), resolution=(1080, 2160)))

touch(Template(r"tpl1603789796067.png", record_pos=(-0.388, 0.666), resolution=(1080, 2160)))

sleep(5)
swipe((527, 2044), (527, 280), duration=3)

sleep(15)

if not exists(Template(r"tpl1603854492318.png", record_pos=(0.007, 0.362), resolution=(1080, 2160))):
    swipe((527, 2044), (527, 280), duration=3)

assert_exists(Template(r"tpl1603854492318.png", record_pos=(0.007, 0.362), resolution=(1080, 2160)), '玩游戏的按钮不存在')

if exists(Template(r"tpl1603873295969.png", record_pos=(-0.006, 0.51), resolution=(1080, 2160))):
    touch(Template(r"tpl1603873313059.png", record_pos=(0.009, 0.509), resolution=(1080, 2160)))
    sleep(5)

touch(Template(r"tpl1603854492318.png", record_pos=(0.007, 0.362), resolution=(1080, 2160)))

sleep(15.0)

assert_not_exists(Template(r"tpl1603798232722.png", record_pos=(0.017, -0.393), resolution=(1080, 2160)),
                  "今日游戏次数已经用完，退出")

touch(Template(r"tpl1603791478271.png", record_pos=(-0.001, 0.165), resolution=(1080, 2160)))

sleep(15.0)

now = time.localtime()
path = os.environ.get('BERSERKER_ENV') + '/meituan'
if not os.path.exists(path):
    os.makedirs(path, exist_ok=True)

file_name = '{}/activity-{}.png'.format(path, time.strftime("%Y%m%d-%H", now))

snapshot(file_name)

if exists(Template(r"tpl1603792766280.png", record_pos=(-0.014, 0.256), resolution=(1080, 2160))):
    touch(Template(r"tpl1603792766280.png", record_pos=(-0.014, 0.256), resolution=(1080, 2160)))
else:
    touch(Template(r"tpl1603855598743.png", record_pos=(-0.007, 0.439), resolution=(1080, 2160)))

sleep(15.0)
file_name = '{}/advert-{}.png'.format(path, time.strftime("%Y%m%d-%H", now))

snapshot(file_name)

stop_app('com.sankuai.meituan')
```

[Airtest官网](http://airtest.netease.com/)：airtest.netease.com
[Airtest教程官网](https://airtest.doc.io.netease.com/)：airtest.doc.io.netease.com
