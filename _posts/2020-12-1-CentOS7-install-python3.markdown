---
layout: post
title:  "CentOS7 安装 python3"
date:   2020-12-01 20:38:35 +0800
categories: python CentOS
---
最近因为迁移系统，而运维离职，只能自己动手进行系统迁移。系统需要迁移到第三方云平台，而又不能使用第三方镜像服务，公司内网镜像服务又不能对外，所以只能打包上传到云平台再进行操作。最终选择了saltstack对多台服务器进行管理，最新的saltstack使用的是python3，由于安装python3遇到了一些坑，所以这里做一些记录。
先查看版本：
```s
python --version
```
显示为2.7

下面安装python3：
```s
yum install python3
```

设置默认python3为默认python需要删除原来python的链接，新建链接python到python3。
```s
rm /usr/bin/python
ln -s /usr/bin/python3 /usr/bin/python
```
到这里python3是安装好了，但是如果执行yum的话会报错，下面处理一下yum的报错问题。

## 修正版本切换产生的错误
这些报错大都是因为使用的是python2，执行所以需要更改shell脚本的配置
1. yum报错
编辑/usr/bin/yum文件，修改python版本
```s
vi /usr/bin/yum
```
更改python为python2.7
```python
#!/usr/bin/python
import sys
```
改为
```python
#!/usr/bin/python2.7
import sys
```

2. 若执行 yum 时出现以下错误：
```s
File "/usr/libexec/urlgrabber-ext-down", line 28
```
执行以下更改,打开该文件并修改首行为：
```s
#!/usr/bin/python2.7
```

3. 执行 yum 时，若出现以下 Error:
```s
Error: Delta RPMs disabled because /usr/bin/applydeltarpm not installed.
```
执行以下安装可解决：
```s
yum install deltarpm
```