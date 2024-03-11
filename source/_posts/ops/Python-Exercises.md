---
layout: post
title: 记一次 Python 脚本坑
date: 2019-10-10 01:09:42
tags:  
    - Python
categories:
  - [运维]
---

## 起因

公司有百来台矿机接入 f2pool 在挖 eth ，但是系统用的并不是我们自己的，也没办法提供权限，所以监控上有一点麻烦。还好 f2pool 也算是比较大的矿池了，看了一下，有提供 api 给用户使用，就写一个监控脚本吧。蛤？ f2pool 还提供微信推送监控？emmmm………别人的哪有自己写的好（其实是因为没有需求也要创造需求（逃

---
## 需求

- 记录掉线矿机 ID 以及掉线时间
- 矿机掉线推送消息到 telegram channel 
- 矿机上线后消息自动删除，防止 channel 内消息过多
---
## 设计

```python   
def main():
    通过 getData() 函数获得推送消息的输出内容以及是否推送的判断，调用 saveLog() 函数并将推送消息内容作为传参，同时将接收系统当前时间的返回值，通过 if 判断消息是否推送，如推送则调用 postMessages()函数开始推送。

def getData(): 
    requests.get 获取访问 api ，通过 decode 方法解码返回值的 content 属性内容，得到完整的 json 数据。然后通过 json.loads 将 json 数据转换成 dict 数据，通过 get 方法取得矿机总数，总算力以及所有矿机详细数据。
        再通过循环取所有矿机的最后一次提交时间，将矿机带时区格式的最后提交时间转换成时间戳格式与当前时间比对，超过十分钟的就算掉线矿机。同时为避免下架矿机频繁推送造成影响，选择掉线超过八小时则默认掉线设备，不予统计。将所有掉线设备计入列表，并综合所有数据产生输出以及是否推送的判断作为返回值。
    
def deleteMessage(messageID):
    接受传递过来的消息 ID ，通过 api 删除该消息。
    
def postMessages(minerData):
    通过api推送消息，并记录返回值，将推送消息 ID 存下，二十分钟后调用 deleteMessage() 函数自动删除该消息，防止 channel 内消息过多，影响查看。
    
def saveLog(minerData):
    接受推送内容参数并记录到 log 文件，同时将当前设备时间作为返回值。
```
----
## 坑


我是比较喜欢新鲜技术的人，俗称小白鼠，所以脚本定时推送选了 systemd timer （其实也并不新，已经沦为主流了），结果推送几乎每两天就挂掉一次，于是开始排查。

第一个版本我写入日志用的是 open() 函数，但是我没有写 close() ，我怀疑是不是文件流打开没有关闭导致的推送卡住，于是我学会了 with-as 语句。然而现实是残酷的，没两天推送又挂了。我开始怀疑是 systemd 的锅，尝试了各种配置方案，无果。最终，偶然在手动调试的时候发现 api 有时候不会返回数据，而且会一直卡在这，我立刻醒悟过来，可能是 requests 拿不到数据反而把自己耽误了，于是加上 timeout 终于了结了。

其实这里我还是有点疑惑，我 systemd 的配置写的是 oneshot ，按照 systemd 的说明，这个选项应该是无论程序执行结果，都会去执行下一次任务，但是却也被卡住了，也许我还没有找到真正的原因……

---
## 代码

```python
#!/bin/env python

import json
import requests
import time
from threading import Timer

ISOTIMEFORMAT = '%Y-%m-%d %H:%M:%S'
LINE = "\n-------------------------------------------------------------------------------------------------------"
DELETETIME = 1190
URL = {'sendMessage': '**YOURTGBOTAPIKEY**/sendMessage',
       'deleteMessage': '**YOURTGBOTAPIKEY**/deleteMessage'}


def getData():
    # 从 f2pool api 获取矿机实时数据
    ethminer = requests.get(
        url='**YOURURL**', timeout=30)
    ethminerDict = json.loads(ethminer.content.decode())
    minerNumber = ethminerDict.get('worker_length')
    minerWorkers = ethminerDict.get('workers')
    minerHashrate = (ethminerDict.get('hashrate')/1000000)

    # 统计掉线矿机 ID 及掉线时间
    dict0 = {}
    for i in range(minerNumber):
        minerTime = (minerWorkers[i][6])
        minerTimeArray = time.strptime(minerTime, "%Y-%m-%dT%H:%M:%S.%fZ")
        minerTimeStamp = time.mktime(minerTimeArray)
        timeNow = time.mktime(time.gmtime(time.time()))
        timeDiff = (timeNow-minerTimeStamp)
        if 28800 > timeDiff > 600:
            dict0[minerWorkers[i][0]] = "已掉线"+str(timeDiff//60)+"分钟"

    # 判断是否有矿机掉线或者算力不正常
    offlineNumber = len(dict0)
    if dict0 or minerHashrate < 30000:
        judge = True
    else:
        judge = False
        dict0 = "无"

    # 回传判断与推送消息
    minerData = "不得了了,矿机掉线了！"+"\r\n"+"目前掉线" + \
        str(offlineNumber)+"台,掉线设备ID为:"+"\r\n"+str(dict0) + \
        "\r\n"+"所有矿机总算力为"+str(minerHashrate)+"MH/s"
    print(judge,minerData)
    return judge, minerData

# 自动删除消息推送
def deleteMessage(messageID):
    deletePost = requests.post(URL['deleteMessage'],
                               data={'chat_id': '-1001241741624', 'message_id': messageID})
    print(deletePost)

# 推送消息
def postMessages(minerData):
    pushPost = requests.post(URL['sendMessage'],
                             data={'chat_id': '-1001241741624', 'text': minerData})
    pushReturn = json.loads(pushPost.text)
    messageID = pushReturn["result"]["message_id"]
    sleepSometime = Timer(DELETETIME, deleteMessage, [messageID])
    sleepSometime.start()

# 保存推送内容
def saveLog(minerData):
    timeNowStr = time.strftime(ISOTIMEFORMAT, time.localtime(time.time()))
    with open('/var/log/push/log', 'a+') as getLog:
        print("现在是UTC时间:", timeNowStr, minerData, LINE, file=getLog)
    return timeNowStr


def main():
    judge, minerData = getData()
    timeNowStr = saveLog(minerData)

    if judge:
        postMessages(minerData)
    else:
        print(timeNowStr, "我安澜，从不掉线")


if __name__ == "__main__":
    main()

```

**ethminer_push.service:**
```profile
[Unit]
Description=ethminer push service
Wants=ethminer_push.timer

[Service]
Type=oneshot
ExecStart=/bin/python "/usr/local/bin/push.py"
```

**ethminer_push.timer:**
```profile
[Unit]
Description=every 10 minute to get

[Timer]
OnBootSec=5min
OnCalendar=*:0/5
Persistent=true

[Install]
WantedBy=timers.target


```

