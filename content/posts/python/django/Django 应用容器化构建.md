---
title: "Django 应用容器化构建"
subtitle: ""
date: 2022-07-20T12:06:37+08:00
lastmod: 2024-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["Django"]
tags: ["Django"]
---
Docker 是一个用于构建、运行和分发应用程序的平台。它允许您将应用程序及其所有依赖项打包到一个容器中，然后可以在任何安装了 Docker 的服务器上运行。

这使得 Docker 成为部署 Web 应用程序的理想选择，因为它可以轻松地将应用程序从一个环境移动到另一个环境，而无需担心任何兼容性问题。

另一方面，Django 是一个 Python Web 框架，可以轻松创建强大且可扩展的 Web 应用程序。 Django 提供了许多开箱即用的功能，例如用户身份验证系统、数据库抽象层和模板引擎。

这使得 Django 的入门变得容易，并且快速、轻松地构建复杂的 Web 应用程序。

Docker 化和部署 Django 应用程序是一个相对简单的过程。涉及的主要步骤是：
1. 为您的 Django 应用程序创建 Dockerfile。
2. 从 Dockerfile 构建 Docker 映像。
3. 将Docker镜像部署到生产环境。
## 准备
基于 Python 3.9、Django 3.2、docker 20.10.13
### 软件安装
- Docker
- Python3
### 创建Django项目
```
pip install django==3.2
django-admin startproject django-demo
```
这行代码将会在当前目录下创建一个 simplesite 目录
让我们看看 startproject 创建了些什么:
```
django-demo/
    manage.py
    django-demo/
        __init__.py
        settings.py
        urls.py
        asgi.py
        wsgi.py
```
这些目录和文件的用处是：

- 最外层的 mysite/ 根目录只是你项目的容器， 根目录名称对 Django 没有影响，你可以将它重命名为任何你喜欢的名称。
- manage.py: 一个让你用各种方式管理 Django 项目的命令行工具。
- 里面一层的 mysite/ 目录包含你的项目，它是一个纯 Python 包。
- mysite/__init__.py：一个空文件，告诉 Python 这个目录应该被认为是一个 Python 包。
- mysite/settings.py：Django 项目的配置文件。
- mysite/urls.py：Django 项目的 URL 声明，就像你网站的“目录”。
- mysite/asgi.py：作为你的项目的运行在 ASGI 兼容的 Web 服务器上的入口。
- mysite/wsgi.py：作为你的项目的运行在 WSGI 兼容的Web服务器上的入口。

如果你的当前目录不是外层的 mysite 目录的话，请切换到此目录，然后运行下面的命令：
```
$ python manage.py runserver
```
## Dockerfile
```
# 使用官方Python运行时作为父镜像
FROM python:3.9-alpine

LABEL maintainer="lushuan2071@126.com"

# 设置 python 环境变量
ENV PYTHONUNBUFFERED 1

# 设置工作目录
WORKDIR /app

# 将代码复制到容器中
COPY . /app

# 添加 pip 清华镜像源
RUN pip install pip -U -i https://pypi.tuna.tsinghua.edu.cn/simple

# 安装依赖
RUN pip install --no-cache-dir -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple

# 暴露端口
EXPOSE 8000

# 运行Django应用
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```
构建镜像
```
docker build -t mydjangoapp:v1.0 . -f Dockerfile
```
运行容器
```
docker run  --name mydjangoapp  -p 8000:8000 -d mydjangoapp:v1.0
```
构建记录
```
# docker build -t mydjangoapp:v1.0 . -f Dockerfile
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
STEP 1/7: FROM localhost/python:3.9-alpine
STEP 2/7: LABEL maintainer="lushuan2071@126.com"
--> bf70fc48b6d
STEP 3/7: WORKDIR /app
--> 45b9db8d767
STEP 4/7: COPY . /app
--> 7df5ef439ea
STEP 5/7: RUN pip install --no-cache-dir -r requirements.txt
Collecting asgiref==3.8.1
  Downloading asgiref-3.8.1-py3-none-any.whl (23 kB)
Collecting Django==3.2.25
  Downloading Django-3.2.25-py3-none-any.whl (7.9 MB)
     ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 7.9/7.9 MB 139.1 kB/s eta 0:00:00
Collecting mysql-connector-python==8.4.0
  Downloading mysql_connector_python-8.4.0-py2.py3-none-any.whl (565 kB)
     ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 565.1/565.1 kB 119.1 kB/s eta 0:00:00
Collecting sqlparse==0.5.0
  Downloading sqlparse-0.5.0-py3-none-any.whl (43 kB)
     ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 44.0/44.0 kB 124.3 kB/s eta 0:00:00
Collecting typing_extensions==4.11.0
  Downloading typing_extensions-4.11.0-py3-none-any.whl (34 kB)
Collecting tzdata==2024.1
  Downloading tzdata-2024.1-py2.py3-none-any.whl (345 kB)
     ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 345.4/345.4 kB 136.4 kB/s eta 0:00:00
Collecting pytz
  Downloading pytz-2024.1-py2.py3-none-any.whl (505 kB)
     ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 505.5/505.5 kB 182.8 kB/s eta 0:00:00
Installing collected packages: pytz, tzdata, typing_extensions, sqlparse, mysql-connector-python, asgiref, Django
Successfully installed Django-3.2.25 asgiref-3.8.1 mysql-connector-python-8.4.0 pytz-2024.1 sqlparse-0.5.0 typing_extensions-4.11.0 tzdata-2024.1
WARNING: Running pip as the 'root' user can result in broken permissions and conflicting behaviour with the system package manager. It is recommended to use a virtual environment instead: https://pip.pypa.io/warnings/venv

[notice] A new release of pip is available: 23.0.1 -> 24.1.2
[notice] To update, run: pip install --upgrade pip
--> dd901d2c952
STEP 6/7: EXPOSE 8000
--> f1884d94f8e
STEP 7/7: CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
COMMIT mydjangoapp:v1.0
--> 9253897f89f
Successfully tagged localhost/mydjangoapp:v1.0
9253897f89fa29c91253cdb28e8494453dd2cf330b66a51968ec817e35faf19b
# docker images|grep django
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
localhost/mydjangoapp                          v1.0           9253897f89fa  13 seconds ago  100 MB
# docker run  --name mydjangoapp  -p 8000:8000 -d mydjangoapp:v1.0
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
49c212881ab19348ccb12f492a7e6f32f545efeff67daf27df6e2b9c7bf074ee
# 
# docker ps
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
CONTAINER ID  IMAGE                       COMMAND               CREATED        STATUS        PORTS                   NAMES
49c212881ab1  localhost/mydjangoapp:v1.0  python manage.py ...  6 seconds ago  Up 7 seconds  0.0.0.0:8000->8000/tcp  mydjangoapp
# docker ps
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
CONTAINER ID  IMAGE                       COMMAND               CREATED        STATUS        PORTS                   NAMES
49c212881ab1  localhost/mydjangoapp:v1.0  python manage.py ...  8 seconds ago  Up 8 seconds  0.0.0.0:8000->8000/tcp  mydjangoapp
```

## 参考
- [Python 镜像的全方位指南 原创](https://blog.51cto.com/jiemei/8712683)

