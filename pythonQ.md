### python 相关

#### 数据结构
- list 实现
- dict 的实现
- set 用法
- 几种数据结构之间的转换

### 迭代器

### 列表解析
- lambda
- filter
- map
- reduce

### 生成器


### 函数
- **args & **kargs
- 函数是对象


### 装饰器

### 类


### flask 的原理
在Flask中，使用werkzeug来做路由分发，werkzeug是Flask使用的底层WSGI库（WSGI，全称 Web Server Gateway Interface，或者 Python Web Server Gateway Interface，是为 Python 语言定义的Web服务器和Web应用程序之间的一种简单而通用的接口）。  
WSGI将Web服务分成两个部分：服务器和应用程序。WGSI服务器只负责与网络相关的两件事：接收浏览器的HTTP请求、向浏览器发送HTTP应答；而对HTTP请求的具体处理逻辑，则通过调用WSGI应用程序进行。
flask 的特点是简单可扩展。简单有几个方面，比如它只实现 web 框架最核心的功能，保持功能的简洁；还有一个就是代码量少，核心代码 app.py 文件只有 2k+ 行。可扩展就是允许第三方插件来扩充功能，比如数据库可以使用 Flask-SQLAlchemy，缓存可以使用 Flask-Cache 等等。
