### Django框架
MTV模式：M 代表模型（Model），T 代表模板（Template），V 代表视图（View）

#### Django 框架的核心工作流程（请求-响应生命周期）。

Django 的 **MVT（Model-View-Template）** 架构中的 **URL -> View -> Template** 链路

1.  **“启动了一个项目，启动了一个app”**
    *   **Django对应点**：Django 最核心的概念就是 `Project`（项目）和 `App`（应用）。开发者通常使用 `django-admin startproject` 创建项目，用 `python manage.py startapp` 创建具体的应用。
2.  **“浏览器发布一个请求过来，先进入urls（路由）”**
    *   **Django对应点**：Django 的 **URL 分发器 (`urls.py`)**。所有进入 Django 的 HTTP 请求，首先会被 Django 的根 URLconf 拦截，并通过正则或路径匹配来寻找对应的路由规则。
3.  **“路由会带着一个视图函数……导向另外一个函数，整个这个函数是做的业务处理”**
    *   **Django对应点**：Django 的 **视图 (`views.py`)**。在 `urls.py` 中，一条路由规则必然绑定一个视图函数（FBV）或视图类（CBV）。请求匹配到路由后，就会执行这个视图函数里的 Python 代码，这里就是处理核心业务逻辑（比如查数据库、计算数据等）的地方。
4.  **“返回一个数据，这个数据是需要html文件渲染的……渲染文件，templates文件”**
    *   **Django对应点**：Django 的 **模板系统 (Templates)**。视图函数处理完业务后，通常会打包一个字典（Context 数据），连同一个指定的 HTML 模板文件（如 `render(request, 'index.html', context)`）交给 Django 的模板引擎。引擎会将数据动态注入到 HTML 文件中（渲染）。
5.  **“将返回值渲染，返回给客户端。这个流程结束。”**
    *   **Django对应点**：生成最终的 HTML 字符串，封装成 `HttpResponse` 对象，通过 Web 服务器（如 WSGI/ASGI）返回给用户的浏览器，完成一次完整的 HTTP 交互。

**补充**
这是一个非常经典的 Web 前后端不分离的 Django 请求流程。它稍微省略了 **Model（模型/数据库）** 的部分（通常是在“业务处理”那个环节去调用 `models.py` 查询数据库）