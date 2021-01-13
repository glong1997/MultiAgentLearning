## 初衷

搞这个东西也不是突发奇想，女朋友最近拉我进了个拼多多群，帮别人砍东西，拿钱。虽然钱不多，但是我们玩得很开心，乐在其中。不过时间长了，有点舍本逐末，想想搞个自动化脚本吧，省时省力。

Java版本刷快手：https://github.com/devospy/shuakuaishou

## 延伸

你也可以使用这些工具，做一下其他事情，例如：抢东西

## 第三方工具

### adb

安卓系统的adb的全称为lAndroid Debug Bridge ，就是起到调试桥的作用，利用adb工具的前提是在手机上打开usb调试，然后通过数据线连接电脑。

将文件名称中含有adb的所有文件复制到C:/windows/system、C:/windows/SysWoW64和C:/windows/SysWoW64目录

```
adb
adb version
```



### uiautomator2

```
pip install --pre uiautomator2
```



### weditor

可视化工具

```
pip install --pre -U weditor
python -m weditor
```

http://localhost:17310/

点击connet，手机这边会安装东西，允许即可。

点击“实时”

### 关于手机

- 在手机“设置”-“关于手机”连续点击“版本号”`7 次`，可以进入到`开发者模式`
- 然后可以到“设置”-“开发者选项”-“调试”里打开USB调试以及允许ADB的一些权限
- 连接时手机会弹出“允许HiSuite通过HDB连接设备”点击允许/接受即可
- 驱动也是必须安装的，可以用豌豆荚，或者是手机商家提供的手机助手，点进去驱动器安装即可（部分电脑双击无法直接进入到驱动器里，可以使用右键找到进入点击即可）

#### 测试连接

```
adb devices
```

测试是否与手机连接成功

### 具体操作

#### 查看应用列表

```
查看所有应用列表：adb shell pm list packages
查看系统应用列表：adb shell pm list packages -s
查看第三方应用列表：adb shell pm list packages -3
```

#### 启动应用

##### Activity

打开`微信`、`淘宝`、快手极速版

```
adb shell am start -n com.tencent.mm/.ui.LauncherUI
adb shell am start -n com.taobao.taobao/com.taobao.tao.welcome.Welcome
adb shell am start -n com.kuaishou.nebula/com.yxcorp.gifshow.HomeActivity
```

##### Service

启动微信的某些服务

```
adb shell am startservice -n com.tencent.mm/.plugin.accountsync.model.AccountAuthenticatorService
```

##### 强制停止

```
adb shell am force-stop com.taobao.taobao
```



#### 模拟按键

```
adb shell input keyevent keycode
```



#### 某些常用功能

截图到电脑

```
adb exec-out screencap -p > sc.png
```

## python

```python
# coding:utf-8
import os

adbShell = "adb shell  {cmdStr}"

def execute(cmd):
    str = adbShell.format(cmdStr=cmd)
    print(str)
    os.system(str)

if __name__ == '__main__':
    # 点击返回按键
    # os.system(" adb shell input keyevent 4 ")
    # 点击
    execute("input tap 928 331")
    # 滑动 从  928 541  滑动到  928 331   用100毫秒
    execute("input swipe 928 541 928 331 100")

    # 点击。 使其获得焦点
    execute("input tap 928 331")
    # 往输入框中输入文字 。前提是输入框获得了焦点
    execute(" input text '1111'")
```



### 淘宝自动浏览

```python
import os
import random
import time
def oneClick(x,y):
    os.system('adb shell input tap ' + str(x) + ' ' + str(y))
def swipe(a,b,c,d):
    os.system('adb shell input touchscreen swipe '+str(a)+' '+str(b)+' '+str(c)+' '+str(d))

def openApp(name):
    os.system('adb shell am start -n ' + str(name))

def click():
    #收猫币
    oneClick(555,1372)
    time.sleep(2)

    oneClick(927,1707)
    time.sleep(1)
    #签到
    oneClick(904,733)
    time.sleep(2)
    #分享活动给好友
    oneClick(899,913)
    time.sleep(4)
    oneClick(103,1291)
    time.sleep(2)
    oneClick(83, 161)
    time.sleep(2)
    oneClick(83, 161)
    
    #浏览特色商店1
    for i in range(0,2):
        oneClick(906,1285)
        time.sleep(2)
        #模拟下滑操作
        swipe(930,880,930,380)
        #等待16s
        time.sleep(22)
        #返回
        oneClick(83, 161)
        time.sleep(2)

    #浏览特色商店2
    for i in range(0,2):
        oneClick(896,1108)
        time.sleep(3)
        #模拟下滑操作
        swipe(930,880,930,380)
        #等待16s
        time.sleep(22)
        #返回
        oneClick(83, 161)
        time.sleep(2)
    
    #浏览18家商店
    for i in range(0,18):
        print("正在浏览第"+str(i+1)+"家商店")
        time.sleep(2)
        #点击查看商店任务
        oneClick(900,1666)
        #模拟下滑操作
        swipe(930,880,930,380)
        #等待16s
        time.sleep(22)
        #返回
        oneClick(83, 161)
        time.sleep(2)
    oneClick(992,400)


if __name__=='__main__':
    
    #打开淘宝app
    openApp('com.taobao.taobao/com.taobao.tao.welcome.Welcome')
    time.sleep(8)
    #打开活动页面
    oneClick(775,1290)
    time.sleep(8)
    click()

    print("ok")
```



### 快速极速版

```python
import time
import subprocess
import random

command_kuaishou = 'adb shell am start -n com.kuaishou.nebula/com.yxcorp.gifshow.HomeActivity'  # 打开快速应用
command_swipe = 'adb shell input swipe 300 600 300 100'   # 滑动视频,四个参数：起始点x坐标 起始点y坐标 结束点x坐标 结束点y坐标。
# command_swipe = 'adb shell input swipe 500 100 500 1400'   # 滑动视频
count = 0
process = subprocess.Popen(command_kuaishou, shell=True)
time.sleep(5)

while(True):
  count = count + 1
  sleep_time = random.randint(10,15)
    # process = subprocess.Popen('adb shell input keyevent KEYCODE_BACK', shell=True)

  process1 = subprocess.Popen(command_swipe, shell=True)
  time.sleep(sleep_time)
  print("第" + str(count) + "次" + "睡眠时间为：" + str(sleep_time) + 'clicks have been completed' )

```

## 更高级点的

```
pip install pocoui
pip install airtest
```



## 参考

[adb下载安装及使用](https://blog.csdn.net/weixin_43927138/article/details/90477966)