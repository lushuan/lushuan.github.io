# Django 模板语法

## Django 模板语法
模板引擎是一种可以让开发者把服务端数据填充到html网页中完成渲染效果的技术。它实现了把前端代码和服务端代码分离的作用，让项目中的业务逻辑代码和数据表现代码分离，让前端开发者和服务端开发者可以更好的完成协同开发。

{{< admonition info >}}
静态网页：页面上的数据都是写死的，万年不变

动态网页：页面上的数据是从后端动态获取的（比如后端获取当前时间；后端获取数据库数据然后传递给前端页面）
{{< /admonition >}}

Django框架中内置了web开发领域非常出名的一个DjangoTemplate模板引擎（DTL）。[DTL官方文档](https://docs.djangoproject.com/zh-hans/3.2/topics/templates/)

要在django框架中使用模板引擎把视图中的数据更好的展示给客户端，需要完成3个步骤：

{{< admonition info >}}

1. 在项目配置文件中指定保存模板文件的模板目录。一般模板目录都是设置在项目根目录或者主应用目录下。
2. 在视图中基于django提供的渲染函数绑定模板文件和需要展示的数据变量
3. 在模板目录下创建对应的模板文件，并根据模板引擎内置的模板语法，填写输出视图传递过来的数据。

{{< /admonition >}}



配置模板目录：在当前项目根目录下创建了模板目录templates. 然后在settings.py, 模板相关配置，找到TEMPLATES配置项，填写DIRS设置模板目录。

```Python
# 模板引擎配置
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [
            BASE_DIR / "templates",  # 路径拼接
        ],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
```

### 简单案例

为了方便接下里的演示内容，我这里创建创建一个新的子应用myapp
```
python manage.py startapp myapp
```
settings.py，注册子应用，代码：
```Python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'myapp',   # 开发者创建的子应用，这填写就是子应用的导包路径
]
```
总路由加载子应用路由,urls.py，代码：
```Python
from django.contrib import admin
from django.urls import path,include
urlpatterns = [
    path('admin/', admin.site.urls),
    # path("路由前缀/", include("子应用目录名.路由模块"))
    path("users/", include("users.urls")),
    path("tem/", include("myapp.urls")),
]
```
在子应用目录下创建urls.py子路由文件，代码如下：
```Python
"""子应用路由"""
from django.urls import path, re_path
from . import views

urlpatterns = [
    path("index", views.index),
]
```
myapp.views.index，代码：
```Python
from django.shortcuts import render
def index(request):
    # 要显示到客户端的数据
    name = "hello DTL!"
    # return render(request, "模板文件路径",context={字典格式：要在客户端中展示的数据})
    return render(request, "index.html",context={"name":name})
```
### render函数内部本质
![django-template-render](/images/python/django-template-render.png)

```Python
from django.shortcuts import render
from django.template.loader import get_template
from django.http.response import HttpResponse

def index(request):
    name = "hello world!"
    # 1. 初始化模板,读取模板内容,实例化模板对象
    # get_template会从项目配置中找到模板目录，我们需要填写的参数就是补全模板文件的路径
    template = get_template("myapp/index.html")
    # 2. 识别context内容, 和模板内容里面的标记[标签]替换,针对复杂的内容,进行正则的替换
    context = {"name": name}
    content = template.render(context, request) # render中完成了变量替换成变量值的过程，这个过程使用了正则。
    print(content)
    # 3. 通过response响应对象,把替换了数据的模板内容返回给客户端
    return HttpResponse(content)
    # 上面代码的简写,直接使用 django.shortcuts.render
    # return render(request, "index.html",context={"name":name})
    # return render(request,"index.html", locals())
    # data = {}
    # data["name"] = "xiaoming"
    # data["message"] = "你好！"
    # return render(request,"index.html", data)
```
1. DTL模板文件与普通html文件的区别在哪里？
DTL模板文件是一种带有特殊语法的HTML文件，这个HTML文件可以被Django编译，可以传递参数进去，实现数据动态化。在编译完成后，生成一个普通的HTML文件，然后发送给客户端。

2. 开发中，我们一般把开发中的文件分2种，分别是静态文件和动态文件。
{{< admonition tip >}}
* 静态文件，数据保存在当前文件，不需要经过任何处理就可以展示出去。普通html文件，图片，视频，音频等这一类文件叫静态文件。

* 动态文件，数据并不在当前文件，而是要经过服务端或其他程序进行编译转换才可以展示出去。 编译转换的过程往往就是使用正则或其他技术把文件内部具有特殊格式的变量转换成真实数据。 动态文件，一般数据会保存在第三方存储设备，如数据库中。django的模板文件，就属于动态文件。
{{< /admonition >}}

### 模板语法
1. 变量渲染（深度查询、过滤器）
```
{{val}}
{{val|filter_name:参数}}
```
2. 标签 tag_name
3. 嵌套和继承
#### 变量渲染之深度查询
```Python
def index(request):
    name = "root"
    age = 13
    sex = True
    lve = ["swimming", "shopping", "coding", "game"]
    bookinfo = {"id": 1, "price": 9.90, "name": "Django 3天入门到挣扎", }
    book_list = [
        {"id": 10, "price": 9.90, "name": "Django 3天入门到挣扎", },
        {"id": 11, "price": 19.90, "name": "Django 7天入门到垂死挣扎", },
    ]
    return render(request, 'index.html', locals())
```
模板代码，templates/index.html：
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    <p>name={{ name }}</p>
    <p>{{ age }}</p>
    <p>{{ sex }}</p>
    <p>列表成员</p>
    <p>{{ lve }}</p>
    <p>{{ lve.0 }}</p>
    <p>{{ lve | last }}</p>

    <p>字典成员</p>
    <p>id={{ bookinfo.id }}</p>
    <p>price={{ bookinfo.price }}</p>
    <p>name={{ bookinfo.name }}</p>

    <p>复杂列表</p>
    <p>{{ book_list.0.name }}</p>
    <p>{{ book_list.1.name }}</p>

</body>
</html>
```
####  变量渲染之内置过滤器
语法：
```
{{obj|过滤器名称:过滤器参数}}

```
内置过滤器

| 过滤器          | 用法                                     | 代码                            |
| :-------------- | :--------------------------------------- | :------------------------------ |
| last            | 获取列表/元组的最后一个成员              | {{liast \| last}}               |
| first           | 获取列表/元组的第一个成员                | {{list\|first}}                 |
| length          | 获取数据的长度                           | {{list \| length}}              |
| defualt         | 当变量没有值的情况下, 系统输出默认值,    | {{str\|default=“默认值”}}       |
| safe            | 让系统不要对内容中的html代码进行实体转义 | {{htmlcontent\| safe}}          |
| upper           | 字母转换成大写                           | {{str \| upper}}                |
| lower           | 字母转换成小写                           | {{str \| lower}}                |
| title           | 每个单词首字母转换成大写                 | {{str \| title}}                |
| date            | 日期时间格式转换                         | `{{ value                       |
| cut             | 从内容中截取掉同样字符的内容             | {{content \| cut:“hello”}}      |
| list            | 把内容转换成列表格式                     | {{content \| list}}             |
| add             | 加法                                     | {{num\| add}}                   |
| filesizeformat  | 把文件大小的数值转换成单位表示           | {{filesize \| filesizeformat}}  |
| `join`          | 按指定字符拼接内容                       | {{list\| join("-")}}            |
| `random`        | 随机提取某个成员                         | {list \| random}}               |
| `slice`         | 按切片提取成员                           | {{list \| slice:":-2"}}         |
| `truncatechars` | 按字符长度截取内容                       | {{content \| truncatechars:30}} |
| `truncatewords` | 按单词长度截取内容                       | 同上                            |

过滤器的使用

视图代码 myapp.views.py;
```Python
def index(request):
    """过滤器 filters"""
    content = "Django template"
    # content1 = '<script>alert(1);</script>'
    from datetime import datetime
    now = datetime.now()
    content2= "hello wrold!"
    return render(request,"myapp/index.html",locals())
```
模板代码,myapp/templates/index.html:
```
{{ content | safe }}
{{ content1 | safe }}

{# 过滤器本质就是函数,但是模板语法不支持小括号调用,所以需要使用:号分割参数 #}
<p>{{ now | date:"Y-m-d H:i:s" }}</p>
<p>{{ conten1 | default:"默认值" }}</p>
{# 一个数据可以连续调用多个过滤器 #}
<p>{{ content2 | truncatechars:6 | upper }}</p>
```
#### 自定义过滤器
虽然官方已经提供了许多内置的过滤器给开发者,但是很明显,还是会有存在不足的时候。例如:希望输出用户的手机号码时, 13912345678 —-» 139*****678，这时我们就需要自定义过滤器。要声明自定义过滤器并且能在模板中正常使用,需要完成2个前置的工作:
```Python
# 1. 当前使用和声明过滤器的子应用必须在setting.py配置文件中的INSTALLED_APPS中注册了!!!
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'myapp',
]


```
myapp.templatetags.my_filters.py代码:
```Python
# 2. 自定义过滤器函数必须被 template.register进行装饰使用.
#    而且过滤器函数所在的模块必须在templatetags包里面保存
   
# 在myapp子应用下创建templatetags包[必须包含__init__.py], 在包目录下创建任意py文件
# myapp.templatetags.my_filters.py代码:

from django import template
register = template.Library()

# 自定义过滤器
@register.filter("mobile")
def mobile(content):
    return content[:3]+"*****"+content[-3:]

# 3. 在需要使用的模板文件中顶部使用load标签加载过滤器文件my_filters.py并调用自定义过滤器
# myapp.views.py,代码:

def index(request):
    """自定义过滤器 filters"""
    
    moblie_number = "13312345678"
    return render(request,"index2.html",locals())
    
```
模板 templates/index2.html 代码: 

注意：以下html 需要首行添加 load my_filters标签处理
```html
# 首行处
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    {{ moblie_number| mobile }}
</body>
</html>
```
#### 标签
##### if
视图代码,myapp.views.py:

```Python
def index(request):
    name = "xiaoming"
    age = 19
    sex = True
    lve = ["swimming", "shopping", "coding", "game"]
    user_lve = "sleep"
    bookinfo = {"id": 1, "price": 9.90, "name": "python3天入门到挣扎", }
    book_list = [
        {"id": 10, "price": 9.90, "name": "python3天入门到挣扎", },
        {"id": 11, "price": 19.90, "name": "python7天入门到垂死挣扎", },
    ]
    return render(request, 'index.html', locals())
```
模板代码,templates/index.html，代码：

路由代码：
```Python
"""子应用路由"""
from django.urls import path, re_path
from . import views

urlpatterns = [
    # ....
    path("index", views.index),
]
```

### 最后
再重申一下模板处理的本质：渲染完成后，生成了字符串，再返回给浏览器。

