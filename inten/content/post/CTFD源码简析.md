---
title: "CTFD源码简析"
date: 2021-06-2T15:07:34+08:00
tags: ["Python", "FLASK"]
ShowToc: true
TocOpen: true
draft: false
---

源地址：[CTFd](https://github.com/CTFd/CTFd)

框架：[Flask](https://flask.palletsprojects.com/en/1.1.x/)

### FLASK

安装与快速启动

https://flask.palletsprojects.com/en/1.1.x/

#### helloword测试


创建hello.py

```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello, World!'
```

```sh
$ export FLASK_APP=hello.py
$ export FLASK_ENV=development  #调试模式
$ python3 -m flask run 
```

#### 官方文档博客

##### 运行

新建项目flaskr

```sh
$ flask init-db        #初始化数据库
Initialized the database.
```


```sh
$ export FLASK_APP=flaskr   #linux
$ export FLASK_ENV=development
$ flask run
```

```sh
# cmd
> set FLASK_APP=flaskr
> set FLASK_ENV=development
> flask run

#PowerShell
> $env:FLASK_APP = "flaskr"
> $env:FLASK_ENV = "development"
> flask run
```

##### 测试

```sh
pytest tests
```

#### 构建和安装

[Deploy to Production — Flask Documentation (1.1.x) (palletsprojects.com)](https://flask.palletsprojects.com/en/1.1.x/tutorial/deploy/)

## 源码路由简析

尝试用flask搭建CTF训练平台，先参照CTFD整理了路由，具体功能的实现还没有分析（在看了在看了

<img src="https://www.kro1lsec.com:442/images/2021/05/25/20210525205629.png" alt="image-20210525205446842" style="zoom:50%;" />

### 根目录：

```python
__init__.py

class CTFdRequest(Request):劫持原始的Flask请求路径
class CTFdFlask(Flask):覆盖的Jinja构造函数设置一个自定义的jinja_environment
class SandboxedBaseEnvironment(SandboxedEnvironment):模仿Flask BaseEnvironment的SandboxEnvironment。
class ThemeLoader(FileSystemLoader):主题结构和配置   
#小收获
#关于import numpy 和 from numpy import *
#不推荐使用import \*，其没有明确地指出你使用了模块中的哪些类。
#并且，如果导入了一个与程序文件中其他东西同名的类，会引发难以发现的错误。
```

```python
auth.py

#确认配置模块
@auth.route("/confirm", methods=["POST", "GET"])
#注册模块
@auth.route("/register", methods=["POST", "GET"])
#登录模块
@auth.route("/login", methods=["POST", "GET"])
#注销模块
@auth.route("/logout")
#修改密码模块
@auth.route("/reset_password", methods=["POST", "GET"])
#配置
@auth.route("/oauth")

challenges.py
#题目信息页
@challenges.route("/challenges", methods=["GET"])

config.py
#具体配置信息

error.py
#意外报错信息

scoreboard.py
#计分板
@scoreboard.route("/scoreboard")

teams.py
#队伍创建，加入，邀请等细节处理
@teams.route("/teams")

user.py
#用户信息
@users.route("/users")

views.py
#项目部署时比赛信息的设置
@views.route("/setup", methods=["GET", "POST"])

config.ini
#配置文件，自带完整的注释
```


### /admin：

```python
__init__.py
#验证管理员身份
@admin.route("/admin", methods=["GET"])

#配置config.html
@admin.route("/admin/plugins/<plugin>", methods=["GET", "POST"])
    
#导入admin配置
@admin.route("/admin/import", methods=["POST"])
#导出admin配置
@admin.route("/admin/export", methods=["GET", "POST"]):
#重置CTFD实例
@admin.route("/admin/reset", methods=["GET", "POST"])

challenges.py
#challenges管理
@admin.route("/admin/challenges")

notifications.py
# 公告管理
@admin.route("/admin/notifications")

pages.py
#新建页面

scoreboard.py
# 记分牌页面
@admin.route("/admin/scoreboard")

statistics.py
# 后台统计数据
@admin.route("/admin/statistics", methods=["GET"])

submissions.py
# 队伍提交记录的页面
@admin.route("/admin/submissions", defaults={"submission_type": None})

teams.py
#队伍信息详情
@admin.route("/admin/teams")

users.py
#用户信息界面
@admin.route("/admin/users")
```

```python
/admin 管理员路由实现
/api   函数功能实现
__init__.py
# 添加命名空间
/cache 缓存清理
/constants
```