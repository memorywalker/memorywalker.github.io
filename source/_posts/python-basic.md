---
title: Python 基础笔记
date: 2021-08-08 09:25:49
categories:
- python
tags:
- python
- Django
---



### Python Crash Course 2nd

基于Python 3.7

python之禅 `import this`

#### 字符串

字符串可以使用`""`或`''`，所以在子串中可以嵌套子串例如

'Messi is the "VIP" winner'。对于字符串还是统一使用`""`来表示，因为有些句子中有`'s`会导致字串匹配错误。

##### 格式化子串

python 3.6支持**f**开始的字串格式化语法，与以前的`full_name = "{} {}".format(first_name, last_name)  `等价

```python
first_name = "ada"
last_name = "lovelace"
full_name = f"{first_name.title()} {last_name.title()}"
```

##### 空白符操作

`"Languages:\n\tPython\n\tC\n\tJavaScript"  `在一句字串中增加换行或tab

```python
favorite_language.rstrip() # 去掉右侧空白
favorite_language.lstrip()
favorite_language.strip()  # 去掉两侧空白
```

#### 数字

指数运算 `3**2` 的值为9

Python在所有需要用到float的地方都会自动转换为float，例如两个整数相除得到的是float

可以在数字间以下划线连接，例如`1_000`，和1000是等价的。(3.6+)

多个变量同时赋值 `x, y, z = 0, 0, 0  `

#### 列表

动态数组，使用[]表示

可以使用负数索引倒序获取列表中的值，例如mylist[-1]，表示获取倒数第一个元素

```python
motorcycles = []
motorcycles[0] = 'ducati' # 修改一个元素
motorcycles.append('ducati') #添加一个元素
motorcycles.insert(0, 'ducati') #插入一个元素
del motorcycles[1]  #删除一个元素
popped_motorcycle = motorcycles.pop() #弹出最后一个元素，并将这个元素赋值给变量
first_owned = motorcycles.pop(0)  # 弹出指定位置的一个元素
motorcycles.remove('ducati')  # 按值删除第一个元素

cars = ['bmw', 'audi', 'toyota', 'subaru']
cars.sort() # 对一个列表升序排序
cars.sort(reverse=True) # 逆序排序
sorted(cars) #对于一个排序，并不改变原来列表的顺序，而是返回一个临时列表
cars.reverse() # 反转列表中所有元素的顺序
len(cars) # 元素个数

#遍历一个列表
for item in list_of_items:
    print(item)
    
# 数字序列
range(5)  # 0-4  
range(1, 5) # [1, 2, 3, 4]
range(2, 11, 2) # 从2开始，步长为2，到11结束，不包括11
even_numbers = list(range(2, 11, 2)) # 序列转列表

digits = [1, 2, 3, 4, 5, 6, 7, 8, 9, 0]
min(digits) # 最小元素
max(digits) # 最大元素
sum(digits) # 元素求和 45
```

##### list comprehension

通过一个列表表达式生成一个列表

`squares = [value**2 for value in range(1, 11)]  `得到

`[1, 4, 9, 16, 25, 36, 49, 64, 81, 100]  `

##### 列表切片

```python
players[1:4] # 获取player列表的1，2，3这3个元素的子集
players[:4] # 从0开始的元素子集
players[2:] # 从2开始到结束的元素子集
players[-3:] # 最后3个元素的子集
mylist = list(range(1, 11))
print(mylist[1:8:3]) # 第三个参数为步长，[2, 5, 8]
friend_foods = my_foods[:] # 拷贝一个新列表，不能用friend_foods = my_foods，这样只是指向同一个列表的另一个别名
```



#### 元组

不可变**immutable** 的列表，`dimensions = (200, 50)  `

```python
my_t = (3,)  # 定义只有一个元素的元组需要多加一个，号
```



#### 表达式

##### boolean表达式

关键字 **True**   **False**

逻辑与 `and`  `(age_0 >= 21) and (age_1 >= 21)  `

逻辑或 `or` `age_0 >= 21 or age_1 >= 21  `

列表中有某一个元素 `'mushrooms' in requested_toppings  `

列表中没有某一个元素 `'mushrooms' not in requested_toppings  `

```python
if a not in words:
    print(a)
elif b in words:
    print(b)
else:
    print("xxx")

# 使用if可以直接判断一个list是否为空
requested_toppings = []
if requested_toppings:
    print(requested_toppings[0])
else:
    print("Empty list")
```





#### 编程规范

Python Enhancement Proposal (PEP)  

PEP 8  说明了编码规范 https://python.org/dev/peps/pep-0008/  

变量一般小写和下划线组成

常量全大写

indent使用空格，不用tab

不要写多余的indent，否则可能出现非预期的结果

```python
magicians = ['alice', 'david', 'carolina']
for magician in magicians:
	print(f"{magician.title()}, that was a great trick!")
	
	print("Thank you everyone!") # 这一行也会被每次循环输出
```

操作符前后各加一个空格`a == b`



#### Django

https://djangoproject.com/  

**开始一个项目之前，一定要写一个项目描述书，包括项目的具体目标，功能，用户交互流程和界面。这样可以保障项目不会偏离，从而正常完成。**

本书中的例子是建立一个学习日志的管理系统

##### 设置开发环境

* 配置一个独立的Python虚拟环境

`python -m venv py38` 会在当前目录下创建一个名为py38的目录，其中是独立的一个python运行环境

* 激活一个虚拟环境
  * windows `py38\Scripts\Activate`  
  * Linux `source py38/bin/activate`

* 安装Django程序库`pip install Django`

##### Django工程

1. 新建一个目录djangoweb，在虚拟环境的终端中，进入这个用来放置工程的目录
2. `(py38) E:\djangoweb>django-admin startproject demo .`在当前目录下新建一个名为demo的工程，注意当前目录的`.`一定要有。
3. 此时会有一个demo工程目录和一个`manage.py`文件在当前目录下
4. 创建数据库 在当前目录下执行`(py38) E:\djangoweb>python manage.py migrate`
5. 测试服务`python manage.py runserver 8000`

manager.py：用来处理管理工程的各种命令，例如迁移数据库，运行服务等

settings.py：django如何与系统交互和管理工程

urls.py：处理URL请求的转发

wsgi.py：web server gateway interfae 用来服务Django创建的文件

* 修改数据库这里都称作migrating the database. 第一次执行migrate命令让django确保数据库和当前工程的状态是匹配的，同时django还会创建一个SQLite数据库文件。

##### app应用

一个Django工程由多个独立的app组成。

重新打开一个虚拟环境终端，切换到工程目录下即manage.py所在的目录，执行

`python manage.py startapp demoapp` 创建一个名称为demoapp的应用。系统会创建这个应用使用的model/view/admin.py文件。

在demo工程目录下settings.py中管理了当前所有应用，在其中可以启用我们自定义的应用

```python
INSTALLED_APPS = [
    'demoapp',
    
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
]
```

自己的app要写在系统默认app之前，可以让自己的app的功能覆盖默认的app的功能。

##### 模型

模型表示数据抽象，和数据库中的一个表对应，例如一本书，它有书名和作者。

一个应用目录中的models.py定义了这个应用的模型。

###### 增加模型

需要在models.py中定义模型的类

```python
# Create your models here.

class Topic(models.Model):
    """A topic"""
    # 少量文字的字段使用CharField，长度限制为200个字符
    text = models.CharField(max_length=200)
    # 使用当前时间作为添加一个Topic的添加时间
    date_added = models.DateTimeField(auto_now_add=True)
	# 这个模型显示时的文字描述信息
    def __str__(self):
        """Return a string representation of the mdoel."""
        return self.text

class Entry(models.Model):
    """Something specific learned about a topic"""
    # 定义一个外键和Topic关联，删除一个Topic时，关联的所有Entry也级联删除
    topic = models.ForeignKey(Topic, on_delete=models.CASCADE)
    text = models.TextField()
    date_added = models.DateTimeField(auto_now_add=True)
	
    # 额外的一些信息用来管理一个模型
    class Meta:
        #告诉Django使用entries来表示多个Entry，如果没有定义这个Django会默认使用Entrys
        verbose_name_plural = 'entries'

    def __str__(self) -> str:
        """Return a string representation of the model"""
        return f"{self.text[:50]}..."
```

这里Topic和Entry作为模型，分别对应了两个数据库表，其中一个Topic和多个Entry关联

###### 更新模型

只要对模型有所修改，即数据表有更改，都需要让Django更新数据表，并进行同步数据库文件。依次执行以下两步：

1. `python manage.py makemigrations demoapp` 会生成类似`demoapp\migrations\0001_initial.py`文件，其中是数据表创建的实现代码
2. `python manage.py migrate`按照数据表的创建代码，更新工程实际的数据库，创建模型对应的数据表

##### Django  管理站点

自动生成的管理员站点，可以管理工程的数据表。需要先创建一个管理员帐号

`python manage.py createsuperuser`执行后，会提示输入用户名和密码，而且密码还有长度要求，但是我输入了123虽然不安全，还是可以继续执行。

```shell
(py38) E:\code\python\djangoweb>python manage.py createsuperuser
Username (leave blank to use 'edison'):
Email address:
Password:
Password (again):
Error: Blank passwords aren't allowed.
Password:
Password (again):
This password is too short. It must contain at least 8 characters.
This password is too common.
This password is entirely numeric.
Bypass password validation and create user anyway? [y/N]: y
Superuser created successfully.
```

打开 http://127.0.0.1:8000/admin/ 使用用户名和密码登录后，就可以看到管理页面，默认会有users和groups两个表. 在这个界面可以直接修改数据表的数据

###### 添加模型到管理站点

在应用的admin.py中增加自己定义的模型

```python
from django.contrib import admin

# Register your models here.

# 当前目录下model模块的Topic和Entry模型
from .models import Topic, Entry

admin.site.register(Topic)
admin.site.register(Entry)
```

##### URL映射

用户访问的url地址通过映射表转给对应的view处理。可以给每个app单独设置一个url映射表。

如果出现`ModuleNotFoundError: No module named  `的错误提示，需要把服务器重新启动一下。

在主工程目录的urls.py中增加app的urls的映射

```python
from django.contrib import admin
from django.urls import path
from django.urls.conf import include

urlpatterns = [
    path('admin/', admin.site.urls),
	# demoapp应用的urls映射，第一个为空，说明从根路径转换
    path('', include('demoapp.urls')),
]
```

在demoapp的目录中新增一个urls.py文件

```python
"""Defines URL patterns for demoapp."""
from django.urls import path # 映射url到views需要用到

from . import views

app_name = 'demoapp'  # Django用来区分同一个工程不同应用的urls.py的文件

urlpatterns = [
    # Home page，第一个参数匹配url相对路径，第二个参数指定调用views.py中的函数，第三个参数给这个url地址起了名字，以便其他地方的代码可以转到这个地址，这样不用写完整的url地址
    path('', views.index, name='index'),
]
```

##### view视图

一个视图函数获取request中的参数信息，处理数据后，将产生的数据发送回浏览器。通常结合模板，将一个页面发送给浏览器。

实现views.py中的index函数

```python
from django.shortcuts import render

# Create your views here.

def index(request):
    """The home page for Demo App."""
    return render(request, 'demoapp/index.html')
```

##### Template模板

模板定义了页面的显示方式，Django把数据填入模板对应的代码片段中。

在demoapp中创建以下目录并创建`index.html`文件`template/demoapp/index.html`这样和view中函数的相对路径保持一致。

###### 模板继承

对于每个页面都有的元素，可以通过定义一个父模板，其中实现通用的界面显示部分，在子模板中继承父模板即可。

* 定义一个父模板`base.html` 其中`{% raw %}xxx{% endraw %}`是为了解决Hexo的nunjunks erro，实际代码不需要

```xml
<p>
    <a href="{ % url 'demoapp:index' % }">Index</a>
</p>
// 定义了一个名为content的block，用来给子模板占位
{ % block content % } { % endblock content % }
```

``{% raw %}{% %}{% endraw %}`定义了一个`Template tag`.这个代码片段用来生成显示在页面上的信息。

`{% raw %} {% url 'demoapp:index' %} {% endraw %}`生成一个URL与`demoapp/urls.py`中的名称为index的url映射匹配，其中的demoapp就是urls.py中定义的**app_name**

* 定义子模板index.html

```xml
{ % extends "demoapp/base.html" % }

{ % block content % }
<p>Learning Log helps you keep track of your learning, for any topic you're
    learning about.</p>
{ % endblock content % }
```

