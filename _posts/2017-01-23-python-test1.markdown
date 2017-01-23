---
layout:     post
title:      "test one note by python"
subtitle:   "this is my first blog"
iframe:     "//alsww.github.io/python-test1/"
navcolor:   "invert"
date:       2017-01-23
author:     "alsww"
tags:
    - python
    <!-- - JavaScript -->
    <!-- - PWA -->
    <!-- - Service Worker -->
---


<!-- > 下滑这里查看更多内容


TLDR; It covers lots of cool stuff about Service Worker!

### [Watching Fullscreen → ](https://huangxuan.me/sw-101-gdgdf/)

<div class="visible-md visible-lg">
    <img src="//huangxuan.me/sw-101-gdgdf/attach/qrcode.png" width="350" />
    <small class="img-hint">Scanning on mobile</small>
</div>



### [Demo Code → ](https://github.com/Huxpro/sw-101-gdgdf)

- Hello World of Service Worker
- Make your own Offline Dinosaurs
- Stale/Fastest while revalidate



### Notes  

This slides is powered by [Yanshuo.io (演说.io)](http://yanshuo.io), a online software helping you create, store and share web slides. 

There are 2 ways that you can fork or contribute this project:

1. `index.html` is the HTML source code exported from [Yanshuo.io](http://yanshuo.io), and many of its dependencis (js, css, fonts) are still linked to CDN of [Yanshuo.io](http://yanshuo.io). You can do any secondary development and host it by yourself.
2. Download the project file under `shuo/`, drag it into [Yanshuo.io](http://yanshuo.io), and you are ready to go. You can edit whatever you want, upload it to your account, and even share your distributions. -->

####celery简介：
celery是一个异步任务队列/基于分布式消息传递的作业队列。它侧重于实时操作，但对调度支持也很好。 celery是用Python编写的，但该协议可以在任何语言实现。更多简介的请自己在网上搜索
 
####环境
系统：centos 7.2 64位
python版本：2.7.x
 
####创建相关的目录
```python  

```
mkdir -p /data/{redis,logs,www}
（我会把python脚本都放在/data/www中）
 
####安装celery
```python
yum install python-devel python-setuptools
easy_install pip
pip install celery
pip install redis   #（python中的redis模块）
 
```

####安装 supervisor

``` python
pip install supervisor
echo_supervisord_conf > /etc/supervisord.conf
```
如果在执行echo_supervisord_conf > /etc/supervisord.conf时报pkg_resources.DistributionNotFound: meld3>=0.6.5错误的话，找到supervisor-3.1.3-py2.6.egg-info/requires.txt，把文件里面meld3 >= 0.6.5注释掉，然后再执行echo_supervisord_conf > /etc/supervisord.conf就好了


####查找方法：
``` python 
find / | grep requires.txt
```
###配置
``` python
vim /etc/supervisord.conf
 修改include配置：
[include]
files = /data/conf/*.ini

之后：
mkdir -p /data/conf/
cd /data/conf/
vim supervisor.ini
内容为：
[program:celery]
command=/usr/bin/celery worker -A tasks
directory=/etc/zabbix/alertscripts   # zabbix报警目录
stdout_logfile=/data/logs/celery.log
autostart=true
autorestart=true
redirect_stderr=true
stopsignal=QUIT
```


 
####安装redis
``` python 
cd /usr/local/src
wget http://download.redis.io/releases/redis-3.0.5.tar.gz
tar xf redis-3.0.5.tar.gz
cd redis-3.0.5
make
make install
make PREFIX=/usr/local/redis install
cp utils/redis_init_script /etc/init.d/redis
chmod a+x /etc/init.d/redis
mkdir /etc/redis
cp redis.conf /etc/redis/
```

 
####简单修改redis.conf（前面数字是行号）
``` python 
42 daemonize yes                                                  #进程转入后台运行
50 port 22222                                                        #修改端口，不用默认的6379
69 bind 127.0.0.1                                                   #绑定IP，只能本地连接
192 dir /data/redis                                                #修改redis文件存放路径
396 requirepass wangwei123      #设置密码
453 maxmemory 256M                                       #redis最大使用256M内存，我的是虚拟机，所以内存设置小
```

``` 注释：
requirepass wangwei123  是配置redis密码，客户端连接和关闭redis时需要此密码
```


####修改/etc/init.d/redis（前面数字是行号）
``` python 
6 REDISPORT=22222
7 EXEC=/usr/local/bin/redis-server
8 CLIEXEC=/usr/local/bin/redis-cli
9 PASSWD="wangwei123"        #增加行
10 PIDFILE=/var/run/redis.pid
11 CONF="/etc/redis/redis.conf"
30 $CLIEXEC -p $REDISPORT -a $PASSWD  shutdown
```

```
备注：
关闭redis时，需要带上redis密码，否则失败，如果没有设置密码，则不需要带密码关闭
```
 
 
####启动redis
``` python 
service redis start
```
 
####查看redis进程
``` python 
ps -ef |grep redis
root     28631     1  0 04:30 ?        00:00:00 /usr/local/bin/redis-server 127.0.0.1:22222
root     28635  1542  0 04:31 pts/0    00:00:00 grep redis
```

 
####编写celery脚本
``` python 

cd /etc/zabbix/alertscripts
vim tasks.py
脚本内容：

#!/usr/bin/env python
#coding:utf-8
#导入celery相关模块、方法
from celery import Celery
from celery import platforms
#导入发送邮件模块
from email.mime.text import MIMEText
from email.header import Header
from email import encoders
from email.utils import parseaddr, formataddr
import smtplib
import json
import datetime
import requests
import pdb
import string
import time
import sys
reload(sys)
sys.setdefaultencoding('utf-8')
#因为supervisord默认是用root运行的，必须设置以下参数为True才能允许celery用root运行
platforms.C_FORCE_ROOT = True
#配置celery，连接redis，wangwei123是redis的密码，22222是端口 10是库（redis有16个库，这取第11个，即10）
config={}
config['CELERY_BROKER_URL'] = 'redis://:wangwei123@127.0.0.1:22222/10'
config['CELERY_RESULT_BACKEND'] = 'redis://:wanwei123@127.0.0.1:22222/10'
#不需要返回任务状态，即设置以下参数为True
config['CELERY_IGNORE_RESULT'] = True
# 定义celery
celery = Celery("tasks", broker=config['CELERY_BROKER_URL'])
celery.conf.update(config)
# 格式化地址
def _format_addr(s):
    name, addr = parseaddr(s)
    return formataddr((Header(name, 'utf-8').encode(), addr))
#异步发邮件
@celery.task
def sendMail(to_list,subject,context):
    from_addr = "alsww@126.com"
    password = "126邮箱的密码"   ##注意这里是126上开启授权码之后的密码，并不是邮箱登陆密码，如果用其他邮箱，根据实际情况配置
    to_addr = to_list
    smtp_server = "smtp.126.com"
    msg = MIMEText(context, 'plain', 'utf-8')
    msg['From'] = _format_addr('Zabbix 监控报警测试：<%s>' % from_addr)
    msg['To'] = _format_addr(to_addr)
    msg['Subject'] = Header(subject, 'utf-8').encode()
    try:
        server = smtplib.SMTP()
        server.connect(smtp_server, 25)
        server.set_debuglevel(1)
        server.login(from_addr,password)
        server.sendmail(from_addr, [to_addr], msg.as_string())
        server.quit()
        return json.dumps({"redid":0,"redmsg":"发送成功"})
    except Exception,e:
        print 'Error: %s' % e
        return json.dumps({"redid":-1,"redmsg":"发送失败"})
```

####编辑异步发送邮件脚本
``` python 
#vim celery_sendmail.py
vim celery_sendmail.py
#!/usr/bin/env python
#coding:utf-8
from tasks import sendMail
#import string
import sys
mail_subject = sys.argv[2]   #主题
mail_to = sys.argv[1]        #收件邮箱
mail_content = sys.argv[3]   #发件内容
print mail_subject
print mail_to
print mail_content
sendMail.delay(to_list=mail_to,subject=mail_subject,context=mail_content)
```

####授权celery_sendmail.py这个脚本再zabbix下的发邮件权限：
``` python 
chmod +x celery_sendmail.py <p>
chown zabbix.zabbix celery_sendmail.py<p>
```

####重启celery服务，并查看celery状态：
``` python 
kill -9 `ps -ef|grep supervisord |grep -v grep |awk '{ print $2 }'` 
/usr/bin/python /usr/bin/supervisord -c /etc/supervisord.conf
ps -ef|grep celery
```

####发邮件测试
>python celery_sendmail.py "alsww@qq.com" "celery异步发邮件测试！" "苦咖啡的测试邮件！" 


到邮箱查看效果：
图片效果：
![Alt text](/Users/ntalker/Documents/markdown-img/mail-result.png)

则说明发送成功。

####之后可以在zabbix上配置脚本发邮件了
注意在zabbix上配置脚本的时候，要传参3个参数：
参数内容为：
{ALERT.SENDTO}
{ALERT.SUBJECT}
{ALERT.MESSAGE}

![Alt text](/Users/ntalker/Documents/markdown-img/mail-result.png)

------------------------------------------------------------


>参考网址：
https://prometheus.io/docs/alerting/alertmanager/
http://www.songjiayang.com/technical/prometheus-with-alertmanager/
https://github.com/vegasbrianc/prometheus
http://www.songjiayang.com/technical/prometheus-with-alertmanager/



