---
title: FastAPI简单使用
date: 2025-08-03 09:10:25
categories:
- python
tags:
- python
- fastapi
- web develop
---

##  FastAPI简单使用

https://fastapi.tiangolo.com/

十几年前上学时候用过Flask，了解了python的WSGI，觉得用它开发web服务很方便。最近了解MCP时发现现在很多python应用都在用FastAPI开发，大概了解了一下，FastAPI是基于python新的ASGI的web框架，它主要利用python的async来实现异步，对于访问量大的web应用效率更高。ASGI可以理解为WSGI的一种进化，它可以通过配置改为WSGI模式。

### 使用场景

* 新开发项目可以直接使用FastAPI，因为它也支持WSGI模式
* 如果是老项目不考虑异步处理请求，只是简单做web应用，还可以用flask

### 使用教程

官方教程 https://fastapi.tiangolo.com/learn/ ，其中

- [Python Types Intro](https://fastapi.tiangolo.com/python-types/) 简单介绍了python的类型系统，现在python 3.6以上版本也支持明确指出参数的类型了

* [Concurrency and async / await](https://fastapi.tiangolo.com/async/) 介绍并发、并行和异步很形象。

#### 安装

1. 使用uv创建一个工程`uv init work-on-fastapi`

2. 进入到`work-on-fastapi`目录下，使用`uv add fastapi[standard] `添加FastAPI依赖

3. uv会自动创建当前工程的虚拟环境，并在虚拟环境中从pypi下载FastAPI

4. 替换main.py中为以下代码测试正常运行

   ```python
   from fastapi import FastAPI

   app = FastAPI()

   @app.get("/")
   async def root():
       return {"message": "Hello World"}
   ```

5. 运行服务 在虚拟环境中执行`fastapi dev main.py`，可以看到提示`Uvicorn running on http://127.0.0.1:8000 `

6. 浏览器打开http://127.0.0.1:8000 确认收到json数据`{"message":"Hello World"}`

7. 打开http://127.0.0.1:8000/docs 可以看到文档页面，或http://127.0.0.1:8000/redoc 看到另一种风格的文档页面，这两个页面可以测试自己的API输入和应答。


#### OpenAPI

OpenAPI 规范（OAS），是定义一个标准的、与具体编程语言无关的RESTful API的规范。OpenAPI 规范使得人类和计算机都能在“不接触任何程序源代码和文档、不监控网络通信”的情况下理解一个服务的作用。

FastAPI使用OpenAPI 规范来定义应用的服务(API)的模式，这里的模式指一个API的路径以及它接收的参数和返回值。这个API模式使用Json数据模式的标准**JSON Schema**来表示。

**Json Schema**定义了一套词汇和规则，这套词汇和规则用来定义Json元数据，且元数据也是通过Json数据形式表达的。Json元数据定义了Json数据需要满足的规范，规范包括成员、结构、类型、约束等。

打开http://127.0.0.1:8000/openapi.json 后会看到以下Json数据

```json
{
  "openapi": "3.1.0",
  "info": {
    "title": "FastAPI",
    "version": "0.1.0"
  },
  "paths": {
    "/": {
      "get": {
        "summary": "Root",
        "operationId": "root__get",
        "responses": {
          "200": {
            "description": "Successful Response",
            "content": {
              "application/json": {
                "schema": {

                }
              }
            }
          }
        }
      }
    }
  }
}
```

#### 程序实现步骤

1. 导入FastAPI模块

2. 创建一个FastAPI实例`app = FastAPI()`这个flask是类似的

3. 通过装饰器顶一个路径操作，例如/，/search，flask里面叫路由。在创建API时，通常使用以下Http方法(OpenAPI中叫做操作Operation)：

   * POST：创建数据
   * GET：获取数据
   * PUT：更新数据
   * DELETE：删除数据

   例如`@app.get("/")`定义了在`/`路径的GET操作

4. 在装饰器下面定义路径操作的处理函数，并返回应答内容

#### 路径参数

可以通过在操作实现函数中说明路径参数的数据类型，这样框架会自动转换数据类型

```python
@app.get("/items/{item_id}")
async def read_item(item_id: int):
    return {"item_id": item_id}
```

请求http://127.0.0.1:8000/items/2.5 会得到错误数据类型的应答

```json
{
  "detail": [
    {
      "type": "int_parsing",
      "loc": [
        "path",
        "item_id"
      ],
      "msg": "Input should be a valid integer, unable to parse string as an integer",
      "input": "2.5"
    }
  ]
}
```

